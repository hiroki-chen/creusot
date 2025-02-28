This is DOWNGRADED version that is compatible with the Rust-SGX-SDK library.

![](/static/marteau.jpg)

*Le marteau-pilon, forges et aciéries de Saint-Chamond, Joseph-Fortuné LAYRAUD, 1889*

# About

**Creusot** is a tool for *deductive verification* of Rust code. It allows you to annotate your code with specifications, invariants and assertions and then *verify* them formally and automatically, returning a *proof* that your code satisfies the specs.

Creusot works by translating Rust code to WhyML, the verification and specification language of [Why3](https://why3.lri.fr). Users can then leverage the full power of Why3 to (semi)-automatically discharge the verification conditions!

See [ARCHITECTURE.md](ARCHITECTURE.md) for technical details.

**Note**: :warning: This tool is mainly for research purposes now. Please tolerate the software quality! :warning:

# Examples of Verification

- [Mutably indexing a linked list](creusot/tests/should_succeed/list_index_mut.rs)
- [Zeroing out a list](creusot/tests/should_succeed/all_zero.rs)
- [Binary search on Vectors](creusot/tests/should_succeed/vector/04_binary_search.rs)
- [Sorting a vector](creusot/tests/should_succeed/vector/02_gnome.rs)
- [IterMut](creusot/tests/should_succeed/iter_mut.rs)

More examples are found in [creusot/tests/should_succeed](creusot/tests/should_succeed).

# Installing

0. Clone the [creusot](https://github.com/xldenis/creusot/) repo at any directory you like
    - Below, we write `REPO` for the (relative or absolute) path to the directory of the repo
1. Set up **Rust**
    - [Install `rustup`](https://www.rust-lang.org/tools/install), to get the suitable Rust toolchain
2. Build **Creusot**
    - Run `$ REPO/build`
3. Set up **Why3**
    - [Get `opam`](https://opam.ocaml.org/doc/Install.html), the package manager for OCaml
    - Specify the [`stackify` branch](https://gitlab.inria.fr/why3/why3/-/tree/stackify) for `why3`: `$ opam pin add why3 https://gitlab.inria.fr/why3/why3.git`
    - Install `why3` and `why3-ide`: `$ opam install why3 why3-ide`
    - Get some SMT solvers: [Z3](https://github.com/Z3Prover/z3) (available by `brew`, `apt`, etc.), [CVC4](https://cvc4.github.io/) (`brew`, `apt`, etc.), [Alt-Ergo](https://alt-ergo.ocamlpro.com/) (`opam`, `apt`, etc.)
    - Configure Why3: `$ why3 config detect`
      * Troubleshoot:
        When your `z3` is a bit too new (e.g., Why3 supports up to ver. 4.8.10 but yours is 4.8.12), Why3 refuses `z3`.
        Then you can try hacking Why3 to make it consider your `z3` be of an older version (e.g., 4.8.10), by updating the relevant field of `~/.why3.conf`.

# Verifying with Creusot and Why3

## Turning to MLCFG

Creusot can translate the Rust programs into a language supported by Why3, called MLCFG (a call flow graph for ML, more specifically WhyML).

By running the following, you can have Creusot turn a Rust program to an MLCFG.
```
REPO/mlcfg PATH/TO/PROGRAM.rs > PATH/TO/OUTPUT.mlcfg
```
The `REPO/mlcfg` command outputs an MLCFG to stdout.
(The file extension of MLCFG is not relevant to Why3.)

You can play with examples in [creusot/tests/should_succeed](creusot/tests/should_succeed).
(There we have `*.stdout` files for the MLCFG outputs.)

Later we show how to write Rust programs with specs for Creusot.

## Proving in Why3

First, in order to process MLCFG outputs by Creusot, you need the [`stackify` branch](https://gitlab.inria.fr/why3/why3/-/tree/stackify) version of Why3, as mentioned in the Installing section. We hope to have this branch integrated and released by 1.5.0 (or ideally earlier).

Now, let's have Why3 process verification conditions of your MLCFG.

You can run the following to call Why3 with the SMT solver Z3 (assuming that you are at the repo dir).
```
REPO/prove -P z3 PATH/TO/OUTPUT.mlcfg
```
You can also change `z3` to `cvc4` or `alt-ergo`.
(`REPO/prove` is a thin wrapper of `why3 prove`.)

You can also run Why3 IDE to view more information or do interactive proofs. You can run the following (assuming that you are at the repo dir).
```
REPO/ide PATH/TO/OUTPUT.mlcfg
```
From there standard proof strategies of Why3 work.

We plan to improve this part of the user experience, but that will have to wait until Creusot gets more stable and complete.

# Writing specs in Rust programs

## Using Creusot for your Rust code

First, you will need to depend on the `creusot-contracts` crate. However, since this crate is not published currently.
To use `creusot-contracts` for your own Rust code, the basic way is to load the crate as an `extern crate`.
You can do that by adding the following declaration to your Rust code:
```rust
extern crate creusot_contracts;
use creusot_contracts::*;
```

:warning: Currently `creusot-contracts` is very unfinished. Using the macros included in this crate may prevent your Rust code from compiling normally. (TODO: implement a pass-through mode for normal compilation) :warning:

## Kinds of contract expressions

Currently Creusot uses 4 different kinds of contract expressions: `requires`, `ensures`, `invariant` and `variant`.

The most basic are `requires` and `ensures`, which can be attached to a Rust function declaration like so:
```rust
#[requires(... precondition ...)]
#[ensures(... postcondition ...)]
fn my_function(i: u32) -> bool { ... }
```
You can attach as many `ensures` and `requires` clauses as you would like, in any order.

Inside a function, you can attach `invariant` clauses to loops, these are attached on *top* of the loop rather than inside, like:
```rust
#[invariant(invariant_name, ... loop invariant ...)]
while ... { ... }
```
Invariants must have names (for now).

## Controlling verification

We also have features for controlling verification.

First, the `trusted` marker lets Creusot trust the implementation and specs.
More specifically, you can put `#[trusted]` on a function like the following:
```rust
#[trusted]
#[ensures(result == 42u32)]
fn the_answer() -> u32 {
  trusted_super_oracle("the answer to life, the universe and everything")
}
```

Also, we have the *unbounded* mode.
This lets Creusot model integer types in Rust as unbounded integers in Why3, suppressing integer overflow checks in Why3.
Currently, this option works only globally, and is enabled by setting the environment variable `CREUSOT_UNBOUNDED` to `1`.
For example, run `REPO/mlcfg` like the following to use the unbounded mode.
```
CREUSOT_UNBOUNDED=1 REPO/mlcfg PATH/TO/PROGRAM.rs > PATH/TO/OUTPUT.mlcfg
```

## Pearlite

Contracts and logic functions are written in Pearlite, a specification language for Rust we are developing. Pearlite can be seen as a pure, immutable fragment of Rust which has access to a few additional logical operations and connectives. In practice you have:
- Base Rust expressions: matching, function calls, let bindings, binary and unary operators, tuples, structs and enums, projections, primitive casts, and dereferencing
- Logical Expressions: quantifiers (`forall` and `exists`), logical implication `==>`, *logical* equality `a === b`, labels
- Rust specific logical expressions: access to the **final** value of a mutable reference `^`, access to the *model* of an object `@`

We also provide two new attributes on Rust functions: `logic` and `predicate`.
Marked  `#[logic]` or `#[predicate]`, a function can be used in specs and other logical conditions (`requires`/`ensures` and `invariant`).
The two attributes have the following difference.
- A `logic` function can freely have logical, non-executable operations, such as quantifiers, logic equalities, etc. Instead, this function can't be called in normal Rust code (the function body of a `logic` function is replaced with a panic).
  You can use pearlite syntax for any part in the logic function by marking that part with the `pearlite! { ... }` macro.
- A `predicate` is a logical function which returns a proposition (in practice, returns a boolean value).

When you write *recursive* `logic` or `predicate` functions, you have to show that the function terminates.
For that, you can add `#[variant(EXPR)]` attribute, which says that the value of the expression `EXPR` strictly decreases (in a known well-founded order) at each recursive call.
The type of `EXPR` should implement the `WellFounded` trait.

You can also give a custom *model* to your type.
To do that, you just implement the `Model` trait (provided in `creusot_contracts`) specifying the associated type `Model`.
You give a trusted spec that defines the model (which can be accessed by `@`) on primitive functions.
For example, the following gives a spooky data type `MyPair<T, U>` a nice pair model.
```rust
impl<T, U> Model for MyPair<T, U> {
    type Target = (T, U);
}
#[trusted]
#[ensures(@result === (a, b))]
fn my_pair<T, U>(a: T, b: U) -> MyPair<T, U> {
  ...
}
```
