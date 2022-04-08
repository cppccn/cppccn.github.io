---
layout: default
title:  "That is not about <i>lazy_static</i>"
description: "
Lazy static in rust can be something strange to immagine. In other languages,
you feel free to do a lot of things, dirty or not I don't judge you, with your
memory...
<a href='./2022/04/07/lazy-or-not-lazy.html'>continue to read</a>"

authors: ["Adrien Zinger"]
reviewers: ["Yvan Sraka"]
---

# I'll not present the lazy_static crate

## Make lazy things yourself

Lazy static in rust can be something strange to think. In other languages,
you feel free to do a lot of things with your program memory, dirty or not I don't judge you. The reason lazy static in rust is quite difficult to imagine, is
the checked lifetime and the coherency of variables.

Don't mess with the rust compiler, there are some rules you have to respect:
1. The type of static should be known at compiletime
2. A static variable should have an initial value
3. If immutable, the variable sould implement the `Sync` trait
4. If mutable, the variable is usable only in unsafe blocks

Start with the beginning, here is a basic example of a static global variable with an interior mutability usable
in a safe block:

```rust
static MY_VAR: Arc<Mutex<(String, bool)>> = Arc::new(Mutex::new((String::new("Hi there"), true)));
```

The `Mutex` give to the global variable a mutability that couldn't exist in a
safe case and the Arc give the `Sync` implementation that allow a
multithreading context required by all immutable static variables.

It doesn't look like a good choice to make lazy things. Even if we use an
Option that we init once and always return the Some value after that. We are
always obligated to lock our static variable to read it. It sounds weird, isn't
it?

I won't present again the `lazy_static` crate. Mr. Tolnay is too much
famous and that post would be anachronistic. ... I just want to know,
why is that lib so powerful?

Going through the library, I found one thing, the heart of the lib. The
[Once](https://doc.rust-lang.org/stable/std/sync/struct.Once.html)
object. The Once is a way to ensure you'll pass once, and only once, in a
function. It's a mega powerful feature of rust. Even if the function you want
to execute is touched by a weird randomization, nondeterministic waits inside,
with multiple calls in parallel threading context, it will be executed _ONCE_.

That's make sens, if you want something lazy_static, as an ip taken from a
setting input, a variable that cannot change during the program life after being initialized, just do that:

```rust
use std::{mem::MaybeUninit, sync::Once};

static mut LAZY_IP: (MaybeUninit<String>, Once) = (MaybeUninit::uninit(), Once::new());
pub fn get_ip(addr: String, port: String) -> &'static String {
    unsafe {
        LAZY_IP.1.call_once(|| {
            let full_addr = format!("{addr}:{port}");
            LAZY_IP.0.write(full_addr);
        });
        &*LAZY_IP.0.as_ptr() // you're sure that it's initialized here
    }
}
```

You probably noticed the usage of [MaybeUninit](https://doc.rust-lang.org/stable/std/mem/union.MaybeUninit.html),
you can prefer the usage of an Option, or a personalized Enum. Whatever your
choice, you can do the initialization with the same method.

You also noticed the usage of the unsafe block, it's because of the
_static mut_, the call_once is here to make the get_ip safe to call in any
context.

Now you know, and so do I, how to make lazy_static things. It wasn't very hard.

## Furthermore

If you want to know how lazy_static can do the magic thing with the
deref star, I'll show you a little example of how to reproduce if without the
crate.

Let's say that we have a function that get our IP from a config file or an env
variable: `find_ip()`. That function would be exactly what you would like to
give in a lazy_static.

```rust
lazy_static::lazy_static! {
    static ref MY_IP: String = find_ip();
}

fn main() {
    println!("My ip is, {}", *MY_IP);
}
```

To get the same result, we just miss a little object, we want to create a
second static structure, that is empty and that implement the deref trait:

```rust
use std::{mem::MaybeUninit, sync::Once};

struct MyIp;

static mut LAZY_IP: (MaybeUninit<String>, Once) = (MaybeUninit::uninit(), Once::new());
static MY_IP: MyIp = MyIp {};

fn get_ip() -> &'static String {
    unsafe {
        LAZY_IP.1.call_once(|| {
            LAZY_IP.0.write(find_ip());
        });
        &*LAZY_IP.0.as_ptr()
    }
}

impl std::ops::Deref for MyIp {
    type Target = String;

    fn deref(&self) -> &Self::Target {
        get_ip()
    }
}

fn main() {
    println!("My ip is, {}", *MY_IP);
}
```

The weakness of using the deref traits is the difficulty to initialize
our lazy variable with a dynamic input. That can be crucial for codes with
dependencies injections. But it's the price we pay to have a beautiful generic
crate!


Thank's for reading, thank you for your time.
