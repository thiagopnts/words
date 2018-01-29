---
layout: post
title: Notes about Rust modules
permalink: notes-about-rust-modules/
---

I've been playing a lot with Rust in the past months and I always get a little confused when trying to use its module system, mostly because some features are not documented yet and has some peculiarities.

The Rust module system has two main components: crates and modules. A crate is a compilation unit and can have as many modules as you want. We're going to focus on how to organize theses modules.

So let's get started, with a Rust environment set up, create a simple project:

$ cargo new rust-modules --bin && cd rust-modules
You can run it and see if everything is fine:

$ cargo run
   Compiling rust-modules v0.0.1
     Running `target/rust-modules`
Hello, world!
We can declare a new module using the mod keyword:

// src/main.rs
fn main() {
    greetings::say_hello();
}

mod greetings {
    pub fn say_hello() {
        println!("Hello");
    }
}
By default, everything inside a module is private, so we have to mark it as public using the pub keyword. We can also have nested modules, with the same encapsulation rules.

We can move our greetings module to a separate file, to keep our main file clean:

// src/greetings.rs
pub fn say_hello() {
    println!("Hello");
}
// src/main.rs
mod greetings;
fn main() {
    greetings::say_hello();
}
As you can see we don't have to declare the module inside our greetings.rs file, the module gets the name of the file. Here starts the trick part, inside our main.rs file we use the mod keyword not to declare a module, but to inform that we are going to import a module. Rust will look for a file called greetings.rs and import its content inside the greetings namespace.

We can change the namespace by using the #[path] attribute and specifing the file by hand:

// src/main.rs
#[path = "greeting.rs"]
mod g;

fn main() {
    g::say_hello();
}
We can also create a directory named greetings and add a file named mod.rs into it. This is useful when you want to create a couple of files related to a module. For instance, lets create two files inside the greetings directory, bar.rs and foo.rs, our structure will look like:

$ tree src/
src/
├── greetings
│   ├── bar.rs
│   ├── foo.rs
│   └── mod.rs
└── main.rs
Inside our bar.rs and foo.rs files we will declare a function:

// src/greetings/bar.rs
pub fn bar_function() {
    println!("Greetings from bar");
}
// src/greetings/foo.rs
pub fn foo_function() {
    println!("Greetings from foo");
}
In order to expose these functions to our greetings module we need change the mod.rs file to import these modules into our greetings module:

// src/greetings/mod.rs
pub mod bar;
pub mod foo;

pub fn say_hello() {
    println!("Hello");
}
Note that we need to make it public so we can use it externally in our main file:

// src/main.rs
mod greetings;

fn main() {
    greetings::say_hello();
    greetings::foo::foo_function();
    greetings::bar::bar_function();
}
This gives us:

$ cargo run
   Compiling rust-modules v0.0.1
     Running `target/rust-modules`
Hello
Greetings from foo
Greetings from bar
But we might want expose the bar_function and foo_function functions directly inside our greetings module, so we can call it directly with greetings::foo_function() and greetings::bar_function(). To do it we need to re-export these functions into our mod.rs file:

// src/greetings/mod.rs

pub use self::bar::bar_function;
pub use self::foo::foo_function;

mod bar;
mod foo;

pub fn say_hello() {
    println!("Hello");
}
Note that the mod bar; and mod foo; imports aren't public anymore, so if we try to run our current main, which uses greetings::foo::foo_function() and greetings::bar::bar_function() were're going to have a function is inaccessible error due to visibility problems. The trick thing here is that we need to pub use the functions we want to re-export before importing the modules. Someone said me on IRC that its a legacy check that the compiler does and requires that the use declarations come before the mod ones.

Now we can update our main to use the new scope:

// src/main.rs
mod greetings;

fn main() {
    greetings::say_hello();
    greetings::foo_function();
    greetings::bar_function();
}
The Rust module system is really powerful and we still have not covered all its features, but I think its a good overview of the most common practices.
