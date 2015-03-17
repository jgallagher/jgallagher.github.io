---
layout: post
title: Using rustc-serialize with Trait Objects
---

> The version of rustc used when writing this post was:

> `rustc 1.0.0-nightly (425297a93 2015-03-11) (built 2015-03-11)`

Recently I've been playing around with a Rust library in which I want to serialize data to JSON, and I ran into some interesting issues with using trait objects.
I've been following Rust without really using it a lot until lately, but I haven't seen much discussion of trait objects.
I'm also curious whether there is a better solution than the one I came up with.

I won't go into much detail about [rustc-serialize](https://crates.io/crates/rustc-serialize) itself.
For that, I'd recommend reading [JSON Serialization in Rust, Part 1](http://valve.github.io/blog/2014/08/25/json-serialization-in-rust-part-1/) and [JSON Serialization in Rust, Part 2](http://valve.github.io/blog/2014/08/26/json-serialization-in-rust-part-2/).
They are a little bit out of date at this point, but not really in meaningful ways (some of the crate and trait names have changed, for example).

## Background

Let's begin with some basic data types and some issues I ran into that more experienced Rustaceans are probably familar with.
Imagine we have a `Document`:

```rust
pub struct Document {
    pub name: String,
    pub contents: String,
}
```

We want to build up a history of all the changes (or deltas) that have been made to any given document.
For the purposes of this post, we'll assume there are two kinds of deltas: setting the document name and appending contents to the end of the document.
We can define a trait and two structs that implement that trait:

```rust
pub trait Delta {
    // Apply this delta to the given Document.
    fn apply(&self, &mut Document);
}

// Concrete Delta type for setting a Document's name
pub struct DeltaSetName {
    name: String,
}

impl Delta for DeltaSetName {
    fn apply(&self, doc: &mut Document) {
        doc.name = self.name.clone();
    }
}

// Concrete Delta type for appending to a Document's contents
pub struct DeltaAppendContents {
    contents: String,
}

impl Delta for DeltaAppendContents {
    fn apply(&self, doc: &mut Document) {
        doc.contents.push_str(&self.contents);
    }
}
```

Finally, we can create a type that holds both a `Document` and a list of all the deltas that have been applied to it.
A first, naive attempt does not compile:

```rust
pub struct DocumentWithHistory {
    doc: Document,

    // Store a vector of Delta trait objects
    history: Vec<Box<Delta>>,
}

impl DocumentWithHistory {
    pub fn new() -> DocumentWithHistory {
        DocumentWithHistory{
            doc: Document::new(),
            history: Vec::new(),
        }
    }

    pub fn apply<T: Delta>(&mut self, delta: T) {
        // apply the delta to our document...
        delta.apply(&mut self.doc);

        // ...and save it in our history
        self.history.push(Box::new(delta));
    }
}
```

The compiler complains with this error message:

```
70:42 error: the parameter type `T` may not live long enough [E0310]
    self.history.push(Box::new(delta));
                      ^~~~~~~~~~~~~~~
70:42 help: consider adding an explicit lifetime bound `T: 'static`...
70:42 note: ...so that the type `T` will meet its required lifetime bounds
    self.history.push(Box::new(delta));
                      ^~~~~~~~~~~~~~~
```

This was my first nontrivial encounter with the `'static` lifetime bound.
What we're trying to do here is take a concrete type `T` that implements `Delta` (for example, `T` might be `DeltaSetName`) and box it up onto the heap as a `Delta` trait object.
The complaint the compiler has here is that since it doesn't know anything about `T`, `T` might itself contain references, and the things it might refer to could disappear while our boxed trait object is still alive (hence the "`T` may not live long enough" message).
The simplest way to address this, as suggested by the compiler, is to add a `'static` lifetime bound to `T`, which means any `T` passed into `apply` cannot contain any non-`'static` references:

```rust
    pub fn apply<T: Delta + 'static>(&mut self, delta: T) {
        // ...
    }
```

Now that we have the basic types created, let's move on to JSON.

## Serialization: Encoding

The end goal is for us to be able to serialize a `DocumentWithHistory` to JSON.
We should be able to run code like this:

```rust
use rustc_serialize::json;
use rustc_serialize::Encodable;
use std::rt::util::Stdout;

fn main() {
    let mut doc = DocumentWithHistory::new();

    doc.apply(DeltaSetName::new("new name".to_string()));
    doc.apply(DeltaAppendContents::new("some contents".to_string()));

    doc.encode(&mut json::Encoder::new(&mut Stdout)).unwrap();
    println!("");
}
```

and see JSON output that includes both the document as it stands and a list of deltas that led to its current state.
Adding serialization support to `Document` and the concrete `Delta` types is easy; we just need to tell the compiler to derive the `RustcEncodable` trait:

```rust
#[derive(RustcEncodable)]
pub struct Document {
    // ...
}

#[derive(RustcEncodable)]
pub struct DeltaSetName {
    // ...
}

#[derive(RustcEncodable)]
pub struct DeltaAppendContents {
    // ...
}
```

(Side note: I'm not sure why the derive name is `RustcEncodable` and the actual trait name is `Encodable`, but that's the current state of things.)
If we try to derive `RustcEncodable` for `DocumentWithHistory`, the error message we get isn't super helpful:

```
57:29 error: type `collections::vec::Vec<Box<Delta>>` does not implement any method in scope named `encode`
 history: Vec<Box<Delta>>,
 ^~~~~~~~~~~~~~~~~~~~~~~~
54:24 note: in expansion of #[derive(Encodable)]
```

Checking [the documentation of Encodable](http://doc.rust-lang.org/rustc-serialize/rustc-serialize/trait.Encodable.html), we can see that `Encodable` is implemented for both `Vec<T>` and `Box<T>` as long as `T` itself is `Encodable`.
The problem here is that our `Delta` trait isn't `Encodable`.
Both our concrete implementations are `Encodable`, so let's try adding that requirement to `Delta`:

```rust
pub trait Delta: Encodable {
    fn apply(&self, &mut Document);
}
```

Now we get a new kind of error:

```
73:42 error: cannot convert to a trait object because trait `Delta` is not object-safe [E0038]
    self.history.push(Box::new(delta));
                      ^~~~~~~~~~~~~~~
73:42 note: method `encode` has generic type parameters
    self.history.push(Box::new(delta));
                      ^~~~~~~~~~~~~~~
```

Adding the `Encodable` requirement to `Delta` means `Delta` is no longer "object-safe".
Huon Wilson [wrote an excellent description of Object Safety](http://huonw.github.io/blog/2015/01/object-safety/); the upshot is that because the `encode` method required by `Encodable` is generic, you cannot reasonably create the vtable that is required to make `Encodable` trait objects.
The free `#[derive(...)]` ride ends here; we need to manually implement `Encodable` for our boxed `Delta`s.

First, let's step back and ask what it means to serialize a `Delta` of some kind known only at runtime.
We (and the compiler) know how to serialize a `DeltaAppendContents` or a `DeltaSetName`, but that won't necessarily be enough information to deserialize.
Imagine we had another delta type, `DeltaSetContents`, that had a single member field named `contents`.
Then if you saw this JSON:

```json
{
    "contents": "Some string."
}
```

should you deserialize that as `DeltaAppendContents` or `DeltaSetContents`?
It's impossible to tell from the encoded fields alone.
We can address this by including a tag in the output describing what kind of `Delta` is being serialized.
We'll add a new enum that we can include in the encoded JSON to identify the delta kind:

```rust
#[derive(RustcEncodable)]
enum DeltaKindTag {
    SetName,
    AppendContents,
}
```

The second thing we need is the ability to downcast, at runtime, from a `Delta` trait object to the actual concrete type.
Arbitrary Rust traits do not (currently?) support downcasting, but with a little help from the `Any` trait and [Chris Morgan's mopa crate](https://github.com/chris-morgan/mopa), we can add downcasting support to `Delta`:

```rust
#[macro_use] #[no_link] extern crate mopa;

pub trait Delta: Any {
    fn apply(&self, &mut Document);
}
mopafy!(Delta);
```

We can now implement `Encodable` for boxed `Delta`s:

```rust
impl Encodable for Box<Delta> {
    fn encode<S: Encoder>(&self, s: &mut S) -> Result<(), S::Error> {
        if let Some(delta) = self.downcast_ref::<DeltaSetName>() {
            (DeltaKindTag::SetName, delta).encode(s)
        } else if let Some(delta) = self.downcast_ref::<DeltaAppendContents>() {
            (DeltaKindTag::AppendContents, delta).encode(s)
        } else {
            panic!("Unknown concrete delta type")
        }
    }
}
```

We step through each of the concrete cases we know about, attempting to downcast.
When we find the one that works, we encode a tuple of the tag identifying the concrete type and the concrete delta itself.
If anyone has suggestions to clean this up, please [file an issue](https://github.com/jgallagher/rust-serialize-trait-objects/issues)!

Unfortunately, this method is pretty gross.
I don't think there's a way to replace the chained `if let`s with a single `match` statement, and in the final `else`, we should really return an error instead of panicing.
(There is [an open issue on rustc-serialize](https://github.com/rust-lang/rustc-serialize/issues/76) that would let us fix the latter, at least.)

Now that `Box<Delta>` is `Encodable`, we can add `#derive[RustcEncodable]` to `DocumentWithHistory` and run our main function from above.
After some prettification, we see exactly what we want:

```json
{
    "doc": {
        "contents": "some contents",
        "name": "new name"
    },
    "history": [
        [
            "SetName",
            {
                "name": "new name"
            }
        ],
        [
            "AppendContents",
            {
                "contents": "some contents"
            }
        ]
    ]
}
```

## Serialization: Decoding

Decoding is quite a bit simpler than encoding.
We don't have to worry about downcasting anything, in particular.
On all the types where we derived `RustcEncodable`, we need to add `RustcDecodable`.
The only tricky bit that remains is implementing `Decodable` for `Box<Delta>`.

Looking back, we encode Deltas as a tuple of their tag and the actual data.
We'll need to decode the tag and then use the correct concrete implementation to decode the data, which means we can't write our implementation like this:

```rust
// DOES NOT WORK
impl Decodable for Box<Delta> {
    fn decode<D: Decoder>(d: &mut D) -> Result<Self, D::Error> {
        let (tag, data) = try!(Decodable::decode(d));
        match tag {
            DeltaKindTag::SetName => /* ... what to do here? ... */,
            DeltaKindTag::AppendContents => /* ... what to do here? ... */,
        }
        // ...
    }
}
```

The compiler cannot determine the type of `data` in the above code.

Instead, we need to use the lower-level methods of `Decoder` to peel out the first element of the tuple and then match on that while reading the second:

```rust
impl Decodable for Box<Delta> {
    fn decode<D: Decoder>(d: &mut D) -> Result<Self, D::Error> {
        // Start reading a 2-long tuple.
        d.read_tuple(2, |d| {

            // Read the first element, our DeltaKindTag.
            let tag = try!(d.read_tuple_arg(0, |d| Decodable::decode(d)));

            // Start reading the second element...
            d.read_tuple_arg(1, |d| {

                // ... by matching on the tag...
                let boxed = match tag {

                    // ... and decoding using the correct concrete implementation.
                    DeltaKindTag::SetName =>
                        Box::new(try!(DeltaSetName::decode(d))) as Box<Delta>,
                    DeltaKindTag::AppendContents =>
                        Box::new(try!(DeltaAppendContents::decode(d))) as Box<Delta>,
                };

                // This closure must return a Result<Box<Delta>, ...>, so wrap up
                // our Box<Delta> in Ok()
                Ok(boxed)
            })
        })
    }
}
```

Hopefully the above code is fairly understandable.
`Decoder` has a number of low level methods for reading types, and each of them takes a closure that has a new `Decoder` for you to use.
One potentially tricky bit is that all the arms of the `match` statement have to return the same type, so we have to add an explicit `as Box<Delta>` to the end of each case.

We can update `main` to encode, then decode, then reencode, and make sure we got the same thing out the second time:

```rust
use rustc_serialize::json;

fn main() {
    let mut doc = DocumentWithHistory::new();

    doc.apply(DeltaSetName::new("new name".to_string()));
    doc.apply(DeltaAppendContents::new("some contents".to_string()));

    let encoded = json::encode(&doc).unwrap();
    println!("encoded = {}", encoded);

    let decoded: DocumentWithHistory = json::decode(&encoded).unwrap();
    let reencoded = json::encode(&decoded).unwrap();

    println!("reencoding matches? {}", reencoded == encoded);
}
```

## Conclusion

A runnable version of all the code in this post is [available on github](https://github.com/jgallagher/rust-serialize-trait-objects).

In the languages I've spent more time with (C++, Objective-C, Swift), I'm not really a fan of inheritance in general.
I'm glad Rust lands firmly on the side of favoring compile-time polymorphism and composition.
However, I think this is a reasonable use case for trait objects, and an interesting exercise in exploring some of the more dynamic features available in the language.

Discuss on [r/rust](http://www.reddit.com/r/rust/comments/2zarra/using_rustcserialize_with_trait_objects/).
