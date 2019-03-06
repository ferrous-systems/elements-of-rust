# elements-of-rust

## Combating Rightward Pressure

### Basics

* use `?` to flatten error handling
* split combinator chains apart when they grow beyond one line

### Tuple Matching

## Lockdown

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
