# :fire: Rust Programming Tipz :fire:

A collection of software engineering techniques for effectively expressing intent with Rust.

# Cleanup

This section is about improving clarity.

## Combating Rightward Pressure

After sparring with the compiler, it's not unusual to stand back and see several nested combinator chains or match statements. Much of the art of writing clean Rust has to do with the judicious application of de-nesting techniques.

### Basics

* Use `?` to flatten error handling, but be careful not to convert errors into top-level enums unless it makes sense to handle them at the same point in your code. Keep separate concerns in separate types.
* Split combinator chains apart when they grow beyond one line. Assign useful names to the intermediate steps. In many cases, a multi-line combinator chain can be more clearly rewritten as a for-loop.

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

If you find yourself writing code that looks like:

```rust
let a = Some(5);
let b = Some(false);

let c = match a {
    Some(a) => {
        match b {
            Some(b) => whatever,
            None => other_thing,
        }
    }
    None => {
        match b {
            Some(b) => another_thing,
            None => a_fourth_thing,
        }
    }
};
```

it can be de-nested by doing a tuple match:

```rust
let a = Some(5);
let b = Some(false);

let c = match (a, b) {
    (Some(a), Some(b)) => whatever,
    (Some(a), None) => other_thing,
    (None, Some(b)) => another_thing,
    (None, None) => a_fourth_thing,
};
```

# Ergonomics

This section is about the mechanical aspects of working with Rust. 

## Unification and Reading the Error Messages That Matter

Rust requires that arguments and return types are made explicit in function definitions. The compiler will use these explicit types at the boundaries of a function to drive type inference. It will take the input argument types and work from the top of the function toward the bottom. It will take the return type and work its way up. Hopefully they can meet in the middle. The process under the hood is actually a [little more complicated than this](http://smallcultfollowing.com/babysteps/blog/2017/03/25/unification-in-chalk-part-1/) but this simplified model is adequate to reason about this particular subject. The point is, there has to be an unbroken chain of type evidence that connects the input arguments to the return type through the body. When there is a gap in the chain, all ambiguous types will turn into errors. This is partially why rust will emit many pages of errors sometimes when there's actually only a single thing that needs to be fixed.

Programming Rust is a long game. To not fall victim to compiler error fatigue, you need to minimize the effort required to deal with compiler errors. A big part of that is to just filter out the errors that don't matter, and usually the most important error to actually fix is the first error that rustc emits. You can use the `cargo watch` plugin to filter out most of these lines, and run the compiler any time you save like this:

```bash
cargo watch -s 'clear; cargo check --tests --color=always 2>&1 | head -40'
```

## Write-Compile-Fix Loop Latency

# Lockdown

This section is about preventing undesirable usage.

### Never

To make a type that can never be created, simply create an empty enum. Use this where you want to prevent compilation of specific codepaths.

```rust
enum Never {}

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
