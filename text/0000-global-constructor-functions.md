- Feature Name: `global_constructor_functions`
- Start Date: 2019-04-18
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Adds an attribute `#[global_constructor]`, which allows a function to be run as part of program initialization, before `main` is called. All global constructors will be run before standard initialization, and in an unspecified order.

# Motivation
[motivation]: #motivation

The main direct motivation behind this is to enable the useful [typetag] crate to function on all platforms.

On a broader scale, this is useful in general for coordination between crates which might not know about each other. typetag is an example of this, but others could be created using the [inventory] crate, or lower level crates like [rust-ctor].

The currently existing mechanism for creating global constructors is [rust-ctor]. It works very similarly to this RFC, but with the disadvantage of only supporting Linux, Windows and Mac. With compiler support, we could extend this to support all platforms.

In particular, the motivation for this RFC stems from wanting support for the the `wasm32-unknown-unknown` platform. See [rustwasm/wasm-bindgen#1216](https://github.com/rustwasm/wasm-bindgen/issues/1216), [mmastrac/rust-ctor#14](https://github.com/mmastrac/rust-ctor/issues/14) and [dtolnay/typetag#8](https://github.com/dtolnay/typetag/issues/8).

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Most rust code runs "after main". Functions running were usually all called at some point called by another function, and those functions were called by others, etc., until we reach the `main` function which started it all.

This makes sense for most cases, but in some cases, having separate entry points which can initialize global structures before main can be useful. Global constructors fill this niche.

Consider this code:

```rust
use std::sync::atomic::{AtomicU8, Ordering};

static my_atomic_var: AtomicU8 = AtomicU8::new(31);

#[global_constructor]
unsafe fn run_this_first() {
    my_atomic_var.store(32, Ordering::SeqCst);
}

fn main() {
    println!("Hello, world! My variable is {}", my_atomic_var.load(Ordering::SeqCst));
}
```

This is an example of declaring a global constructor. `run_this_first` will be called during program initialization, separately from main. When multiple global constructors exist, it's unspecified which runs first.

At compile time, we initialize `my_atomic_var` to `31`. Also at compile time, the list of global constructors is created, and includes our function `run_this_first`.

When our program runs, first, the global constructors are loaded, and run in some order. Among these is `run_this_first`, and `my_atomic_var` is set to `32`.

Finally, we enter `main`, and observe the variable as `32`.

Great, right? It is, but we have to be careful.

Global constructors don't just run before main, they also run before other initialization the standard library performs. They should never use `std::io`, and should never panic. To recognize this unsafety, all `global_constructor` functions must be declared as unsafe.

Finally, note that creating global constructors should be avoided whenever possible. Any nontrivial computation can slow down program startup, and upmost care must be taken to ensure all code is sound.

Most rust code, outside of a few support libraries, is expected to be completely free of global constructors. If this kind of initialization is needed, it's recommended to use a higher level library like [inventory]. Other crates might use this without ever realizing it, when they use crates like [typetag].

As using global_constructor directly is discouraged, I don't anticipate this feature being taught to new rust programmers.

Constructing things at compile time  and running code at runtime is
Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

For implementation-oriented RFCs (e.g. for compiler internals), this section should focus on how compiler contributors should think about the change, and give examples of its concrete impact. For policy RFCs, this section should provide an example-driven introduction to the policy, and explain its impact in concrete terms.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Any otherwise-plain unsafe rust function may be marked with the `#[global_constructor]` attribute. When marked, it will be added to a list internal to the compiler, and included as a global constructor to be run before main on program launch.

For example,

```rust
#[global_constructor]
unsafe fn my_constructor() {}
```

Internally, this will add the function to LLVM's [`@global_ctors`](https://llvm.org/docs/LangRef.html#the-llvm-global-ctors-global-variable) list.

Global constructor functions will be allowed in all crates, and in all modules (including inside other functions). Privacy of the global constructor function will not effect it being a global constructor.

Global constructor functions must:
- take zero arguments
- be declared as unsafe
- return `()`

To mark a function as `#[global_ctor]` without satisfying these requirements is an error

All global constructor functions in all included crates will be called, but the order is explicitly unspecified. There are no guarantees, even for functions declared directly adjacent in the same module.

Global constructor functions may be under a `#[cfg]` flag, and this will behave as expected. The following constructor will be called if the code is compiled with the `construct_things` feature, and won't if it isn't.

```rust
#[cfg(feature = "construct_things")]
#[global_constructor]
unsafe fn my_constructor() {}
```

Global constructor functions must not be under `#[target_feature]`. Global constructor functions are always called, and thus it wouldn't make sense to have code which can't run on some of the target machines. For example, both of the following will result in compile-time errors:

```rust
#[target_feature(enable = "sse3")]
#[global_constructor]
fn bad_ctor() {
}
```

```rust
#[target_feature(enable = "sse3")]
mod my_sse3_module {
    #[global_constructor]
    fn second_bad_ctor() {
    }
}
```

# Drawbacks
[drawbacks]: #drawbacks

This introduces life-before-main into rust, something we explicitly avoided in the early days. See this quote from the [old website's FAQ](https://prev.rust-lang.org/en-US/faq.html#does-rust-allow-non-constant-expression-values-for-globals) (inlined for posterity):
> #### Does Rust allow non-constant-expression values for globals?
> No. Globals cannot have a non-constant-expression constructor and cannot have a destructor at all. Static constructors are undesirable because portably ensuring a static initialization order is difficult. Life before main is often considered a misfeature, so Rust does not allow it.
>
> See the [C++ FQA](http://yosefk.com/c++fqa/ctors.html#fqa-10.12) about the “static initialization order fiasco”, and [Eric Lippert’s blog](https://ericlippert.com/2013/02/06/static-constructors-part-one/) for the challenges in C#, which also has this feature.

This brings up two problems which are still relevant today:

- Like C++'s static initializers, global constructors will have an unspecified run order. As we can't initialize (only change) statics in global constructors, the danger is somewhat mitigated, but not entirely. Two global constructors could still rely on run order, and introduce subtle bugs.

- This feature allows users to interact with `std` before it's been properly initialized. Anyone not careful could easily either break `std`'s invariants, or create code which silently depends on any given part of `std` not requiring initialization.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Only allow `#[global_constructor]` on WASM until a future expansion.

  My personal motivation stems from wanting this on WASM, and it could serve as a single platform to test things out on before either removing the feature or promoting it to all platforms.

  This has the disadvantage of arbitrarily reducing what platforms are supported, while it's likely that LLVM would be able to support many more current platforms (given that they support static initializers in C++).
- Rename the attribute.

  C++ calls this kind of thing a "static initializer", and the existing [rust-ctor] crate simply uses the `#[ctor]` attribute.

  We could rename `#[global_constructor]` to `#[global_ctor]`, `#[ctor]`, `#[initializer]`, `#[static_initializer]`, or another combination.

  I chose `#[global_constructor]` as it's reasonably descriptive, references being program-wide, and doesn't imply it's necessarily initializing anything. It likely is initializing something, but it's a valid use of the feature to have a global constructor which performs no initialization whatsoever.
- Apply `#[global_constructor]` to `unsafe fn()` typed statics, in addition to or instead of to functions.

  This strictly increases the possible uses. I've not included it for the sake of being minimalistic, no other reason.
- Simply don't implement this.

   As mentioned in the drawbacks section, this is a dangerous addition.

  If we don't implement this, we could then further shame using `rust-ctor`, or let it be and simply not "grace" the feature with compiler support.

  On the other hand, I believe the advantages do outweigh the disadvantages. We currently don't have a good way to implement cross-crate initialization (like [inventory]) without global constructors. If we explicitly discourage direct use and ensure all users are aware of the unsafety, we should be able to minimize the danger.
- Somehow define the order in which global constructors are called.

  For example, in C++, it's guaranteed that static initializers declared in the same file execute in the same order that they are declared in. It could be prudent to define partial order, or a complete order, to the way in which rust's global constructors run.

  This could be useful, but it could also allow people to start depending on fickle orders. The order being unspecified leaves it all up in the air, and ensures that users know they can't depend on anything they don't synchronize themselves.

# Prior art
[prior-art]: #prior-art

This is heavily inspired by the [rust-ctor] crate, which implements this feature in "user-land" for Linux, Windows and Mac.

The non-Rust main prior art I know of is the feature "static initializers" in C++. A number of blogs describe using this feature, and why not to:
- [What’s the “ `static` initialization order ‘fiasco’ (problem)”? (isocpp.org)](https://isocpp.org/wiki/faq/ctors#static-init-order)
- [C++ Static initialization is powerful but dangerous](http://cplusplus.bordoon.com/static_initialization.html)
- [Static initializers will murder your family (somewhat satirical)](https://meowni.ca/posts/static-initializers/)

C++ static initializers allow a program to initialize static variables to some value, possibly calling functions which will be run at runtime. Problems occur when one static variable depends on calling methods on another: the initialization order is unspecified, so programs written this way crash 50% of the time.

This could definitely be a problem in Rust as well, but it's mitigated by the fact that `#[global_constructor]` will never initialize a static variable. It can change the value, but all statics must still have some sane and allowed default. The second mitigation will be heavily discouraging use of this feature outside of specific abstraction libraries, such as the aforementioned [inventory] crate.


# Unresolved questions
[unresolved-questions]: #unresolved-questions
- Is it feasible to allow panicking within global constructors?

  I personally don't know enough about landing pads and such to know if we could do this. (or even if we could say something like "all panics in global constructors are guaranteed segfaults", which would be better than leaving this up in the air).
- Should static methods be allowed as global constructors? For example,
  ```rust
  impl MyStruct {
      #[global_constructor]
      fn my_gctor() { ... }
  }
  ```
- Should extern functions be allowed as global constructors? For example,
  ```rust
  #[global_constructor]
  unsafe extern "C" fn my_gctor() { ... }
  ```
- What platforms is it feasible to support?

  I make the assumption that because C++ has static initializers, LLVM will support `@global_ctors` on all of its supported platforms. This could be a naive assumption.
# Future possibilities
[future-possibilities]: #future-possibilities

- If we limit this to some platforms, it could later be expanded to all platforms rust supports.
- `#[global_constructor]` on `unsafe fn()`-typed statics could be added at a later date.
- The order in which global constructors are run could be partially or fully defined at a later date.

[inventory]: https://github.com/dtolnay/inventory/
[typetag]: https://github.com/dtolnay/typetag
[rust-ctor]: https://github.com/mmastrac/rust-ctor/
