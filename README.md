# :fire: Rust Programming Tipz :fire:

A collection of software engineering techniques for effectively expressing intent with Rust.

# Cleanup

This section is about improving clarity.

## Combating Rightward Pressure

Code can look 

### Basics

* use `?` to flatten error handling
* split combinator chains apart when they grow beyond one line

### Using Blocks

Next time you need to spawn a `move` closure, remember that blocks are
expressions. Seen in
[salsa](https://github.com/salsa-rs/salsa/blob/3dc4539c7c34cb12b5d4d1bb0706324cfcaaa7ae/tests/parallel/cancellation.rs#L42-L53) and described in more detail in [Rust pattern: Precise closure capture clauses](http://smallcultfollowing.com/babysteps/blog/2018/04/24/rust-pattern-precise-closure-capture-clauses/#a-more-general-pattern).

```
// Before
fn spawn_threads(config: Arc<Config>) {
    let config1 = Arc::clone(&config);
    thread::spawn(move || {
        do_x(config1);
    });
    let config2 = Arc::clone(&config);
    thread::spawn(move || {
        do_y(config2);
    });
}

// After, no need to invent config_n names
fn spawn_threads(config: Arc<Config>) {
    thread::spawn({
        let config = Arc::clone(&config);
        move || {
            do_x(config);
        }
    });
    thread::spawn({
        let config = Arc::clone(&config);
        move || {
            do_y(config);
        }
    });
}
```

### Tuple Matching

# Ergonomics

This section is about the mechanical aspects of working with Rust. 

## Unification and Reading the Error Messages That Matter

## Write-Compile-Fix Loop Latency

# Lockdown

This section is about preventing undesirable usage.

### Never

To make a type that can never be created, simply create an empty trait. Use this where you want to prevent compilation of specific codepaths.

```rust
trait Never {}

let never = Never:: // oh yeah, can't actually create one...
```

### Making traits unimplementable

If you want to prevent others from implementing your Trait, use the following pattern to "seal" it so that only your implementations will ever exist. Seen in tokio-tls.

```rust
mod sealed {
    pub trait Sealed {}
}

// must have access to sealed::Sealed to
// implement this trait, which is not possible
// for other crates etc...
pub trait MyPublicTrait: sealed::Sealed {}

pub struct MyStruct;

impl sealed::Sealed for MyStruct {}

impl MyPublicTrait for MyStruct {}
```

### Deactivating Mutability
Here's a pattern for disabling mutability for "finalized" objects, even in mutable owned copies of a thing, preventing misuse. Done by wrapping it in a newtype with a private inner value that implements Deref but not DerefMut:

```rust
mod config {
    #[derive(Clone, Debug, PartialOrd, Ord, Eq, PartialEq)]
    pub struct Immutable<T>(T);

    impl<T> Copy for Immutable<T> where T: Copy {}

    impl<T> std::ops::Deref for Immutable<T> {
        type Target = T;

        fn deref(&self) -> &T {
            &self.0
        }
    }
    
    #[derive(Default)]
    pub struct Config {
        pub a: usize,
        pub b: String,
    }
    
    impl Config {
        pub fn build(self) -> Immutable<Config> {
            Immutable(self)
        }
    }
}

use config::Config;

fn main() {
    let mut under_construction = Config {
        a: 5,
        b: "yo".into(),
    };
    
    under_construction.a = 6;
    
    let finalized = under_construction.build();
    
    // at this point, you can make tons of copies,
    // and even if somebody has an owned local version,
    // they won't be able to accidentally change some
    // configuration that
    println!("finalized.a: {}", finalized.a);
    
    let mut finalized = finalized;
    
    // the below WON'T work bwahahaha
    // finalized.a = 666;
    // finalized.0.a = 666;
}
```

### Box<FnOnce>

Currently, it's not possible to call `Box<FnOnce(T) -> R>` on stable Rust. The
common workaround is to use `Box<FnMut(T) -> R>`, store internal state inside of
an `Option` and `take` the state out (with a potential run-time panic) in the
call. However, a solution that statically guarantees that fn can be called at
most once is possible. Seen in [Cargo](https://github.com/rust-lang/cargo/blob/dc83ead224d8622f748f507574e1448a28d8dcc7/src/cargo/core/compiler/job.rs#L17-L25).

```rust
trait FnBox<A, R> {
    fn call_box(self: Box<Self>, a: A) -> R;
}

impl<A, R, F: FnOnce(A) -> R> FnBox<A, R> for F {
    fn call_box(self: Box<F>, a: A) -> R {
        (*self)(a)
    }
}

fn demo(f: Box<dyn FnBox<(), String>>) -> String {
    f.call_box(())
}

#[test]
fn test_demo() {
    let hello = "hello".to_string();
    let f: Box<dyn FnBox<(), String>> = Box::new(move |()| hello);
    assert_eq!(&demo(f), "hello");
}
```

Note that `self: Box<Self>` is stable and object-safe.
