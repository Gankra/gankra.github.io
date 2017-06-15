% The Pain Of Real Linear Types in Rust

For whatever reason "linear" types in Rust came up at work today, at which point I made my usual assertion that they'd be a nightmare to implement, because they don't compose well. I tried to explain it off the top of my head, but I figured it's best for me to just write it out.



# Definitions And The State Of Rust

So there's this thing called a [Substructural Type System](https://en.wikipedia.org/wiki/Substructural_type_system)™, that Rust takes some ideas from. It's poorly named, and so are most of the concepts it introduces. Just so you can look this stuff up in The Literature, I'll be providing all these bad names with a Trademark of Disdain™, but otherwise I'd prefer to use Rust-centric terminology.

So this type system defines three operations that a (instance of a) type might support. Two of them are interesting to us, and one of them we don't care about. Here's the two we care about:

* can be used less than once (weakening™)
* can be used more than once (contraction™)

The definition of use is purposefully vague here, because as you'll see Rust plays around with "what is a use" a lot to make the type system do things we actually care about. 

All the combinations of having-and-not-having these properties gives us 4 interesting kinds of type:

1. can be used any number of times (no name - the default)
2. can't be used more than once (affine™)
3. must be used at least once (relevant™ - this one is a decent name)
4. must be used exactly once (linear™)

Also for added confusion, sometimes linear™ or affine™ is used as a synonym for the whole substructural™ system. Nice™.




## Can Be Used Any Number Of Times 

This one doesn't really need any justification -- it's how basically every programming language works. Rust provides this semantic in two forms: Copy and Clone. Copy is the form we're all used to: total unrestricted use of the value. In Rust it has a very specific language semantic as well: copying the value can be done with a bitwise copy, and discarding the value can be done by just forgetting about it (it can't implement Drop). 

This is in contrast to Swift and C++, which may implicitly execute code when a value is copied or destroyed (e.g. incrementing and decrementing reference counts).

Clone is like Copy, but where we might need to do some work when we do the copy or destroy -- like the aforementionned reference count manipulation. Copying a Clone value requires an explicit call to `clone()`, while destroying a Clone value implicitly calls `drop()`. [^drop]

Plain Old Data types like `i32`, `f64`, and `[i32; 4]` are Copy. Almost every type in Rust is Clone (or, at least, could be without any issue).





## Can't Be Used More Than Once

This is what Rust calls move semantics, or move-only types. e.g.

```rust
let x = vec![1, 2, 3];
let y = x;           // NOTE: x moved here
println!("{:?}", x); // ERROR: use of moved value x
pritnln!("{:?}", y); // OK 
```

Here Rust defines a "use" to be pass-by-value. Pass-by-reference isn't considered a use, because we want a way to *actually* make use of these value more than once. The borrow checker makes sure we get rid of all these references before we perform a "real" use. [^pass-by]

The purpose of this system is that it gives us a simple way to prove that we have unique access to a value. If you have a move-only type by-value, you know that no one else is looking at it, and it's safe to do something that invalidates it, like freeing a pointer it owns. This also gives fairly strong aliasing guarantees for mutable references, which helps avoid tons of logic bugs.

More broadly, this system can be used as a *proof of work*. For instance, if I need step1() to always precede step2(), I can use a move-only value to enforce this:

```rust
fn step1() -> Step1Token;             // malloc, fopen, glCreateContext, connect
fn step2(Step1Token) -> Step2Token;   // write, read, mutate, draw, send
fn step3(Step2Token);                 // free, close, destroy
```

We have to do these steps in order, and we can't repeat any steps we shouldn't (references let us easily make repeatable steps):

```rust
let token1 = step1();
let token2 = step2(token1);
step3(token2);
step2(token1); // ERROR: use of moved value
step3(token2); // ERROR: use of moved value
```

Although if your API has hidden global state, you can still have some bugs:

```rust
let token1a = insert_start();
let token1b = insert_start();      // clobbers first insert_start
let token2a = write_data(token1a); // actually writes to token1b's buffer
insert_end(token2a);               // closes token 1b's buffer
write_data(token1b);               // who knows what this will do
```

Most types don't have this problem though -- they own all the context they care about.

Another bug move-only types can't prevent is *forgetting to do a thing*. We can make sure you perform step1 before step2, but we can't actually make you do step2. Which brings us to the next kind of type...





## Must Be Used At Least Once

This is the form Rust has the loosest support for, but it does have *some*. Aspects of At Least Once can be seen in two places:

* the unused_variables, unused_assignments, and unused_must_use lints
* Drop

unused_variables is the simplest form. Lots of compilers support it, since it's pretty easy to implement and strongly indicative of an implementation bug.

```rust
fn main() {
    let x = 0; // WARNING: unused variable `x`
    let y = 1;
    if y == y { println!("OK!") }
}
```

Here a use is defined much more weakly: basically any use of the identifier `x` will silence the lint:

```rust
fn main() {
    let x = 0;
    let _ = x; // OK!
}
```

unused_assignments is similar, but instead checks to see if there are any assignments that always get overwritten before a read:

```rust
fn main() {
    let mut x = 0; // WARNING: value assigned to `x` is never read
    x = 1;
    println!("{}", x);
}
```

must_use is an annotation that can be applied to a type or a function, to indicate that the function, or any function that returns a must_use type, shouldn't have its return value implicitly ignored. Result is the most notable must_use type:

```rust
fn read() -> Result<i32, i32> { Ok(0) }

fn main() {
    read();         // WARNING: unused result which must be used
    let _ = read(); // OK
}
```

All of these together are generally good enough to remind users to call step2() after step1(), especially if step2 is usually immediately after step1. However in more complex scenarios it's easy to accidentally mark the value as "used" without actually properly using it. For instance, printing a Step1Token or putting it in a collection will mark it as used for the purposes of these analyses.

Drop provides stronger support for must-use values by introducing a final step that will automatically be called on a value as it's destroyed. Assigning to a variable, printing it, stuffing it in collection, or even calling panic!() is no escape: drop() will be called.

Actually there's one escape hatch: mem::forget(). mem::forget will take any value and prevent it from being Dropped. It can be thought of as The All User of values, that any must-use value can be used by. Mostly it's just used by unsafe code to take ownership of the data owned by a Drop type (see: Vec::into_raw_parts).

Also there's some several ways for destructors to not run (often called "destructor leaking"):

* building a reference counted cycle which will leak into infinity
* overwriting the value with ptr::write/copy
* aborting the program
* never leaving the scope the value is defined for
* &lt;vague mumbling about double panicking&gt;


Drop also has several annoying limitations:

* It can't produce a value
* It can't take extra values in
* There Is Only One Drop

This means that Drop can't be used to ensure step2 gets called, nor can it be used if `step3(takes, other, values)`. It also means it can't be used to ensure one of step3a or step3b is called.

One extreme option that I've seen is to implement drop() as `abort("this value must be used")`. All "proper" consumers then mem::forget the value, preventing this "destructor bomb" from going off. This provides a *dynamic* version of strict must-use values. Although it's still vulnerable to the few ways destructors can leak, this isn't a significant concern in practice. Mostly it just stinks because it's dynamic and Rust users Want Static Verification.

Ultimately, Rust lacks "proper" support for this kind of type.

(Edit: a commenter pointed out to me that you can hack some stronger must-use guarantees by making step3 return a value, and having the current function return a Step3Token. This makes it so that it's impossible to return from the function without a proof of completing step3. But there's lots of holes in this design, not the least of which is that the user of your API needs to be "in on it" and set that up).

(Edit2: It's possible for a library to statically require its client to call functions up to and including step3 by providing `with_step1<R, F: FnOnce(Step1Token) -> (R, Step3Token)>(f: F) -> R` instead of `step1() -> Step1Token`, which fixes the first edit's approach's issue of require the client to have prior knowledge of step3. This is a similar pattern to what `scoped_threadpool` uses to work around the [JoinGuard unsoundness](https://github.com/rust-lang/rust/issues/24292))

## Must Be Used Exactly Once

There's nothing actually interesting about this case. It's just a move-only must-use type. People who say they want Linear™ Types In Rust actually just want Proper Support For Relevant™ Types.

As such, we will focus on what providing must-use types looks like.






# Adding Proper Must-Use Types To Rust 

Proper support for must-use types requires three things:

* a checker that ensures a must-use value is always locally moved (mem::forget being the ultimate destination of all must-use types)
* a proper trait bound for code to support and derive must-useness with
* stdlib support

I posit that this system is deeply unpleasant.




## The Checker

The checker is trivial to implement. It already basically exists in rustc to implement dynamic drop flags; it's just a basic definite-initialization analysis. The only tweak that needs to be made is to produce an error if a must-use type isn't definitely uninitialized by the end of the function, or overwritten while already initialized. There would need to be some special handling if `<T: Copy>`, I think this is easy but wouldn't be surprised if weird stuff comes up.

I'm pretty sure you don't need to do anything special for closures that take must-use values as arguments. The creator of the closures just checks that the closure body moves the input, like any other function.

I don't think anything at all needs to be done with the borrow checker? There's probably some new must-use reference type you could make but I can't think of what the point of that would be (it can't be &uninit, because must-use types can be passed to mem::forget).

(Edit: a commenter pointed out that instead of mem::forget as The Final Sink, you could use *destructuring*. This would basically give anyone with permission to access all of a type's fields the right to mark a must-use type as fully used. With this you could potentially make &uninit useful.)

Some of the holes that Drop has would remain in effect:

* building a reference counted cycle which will leak into infinity 
    * solution?: Rc/Arc don't opt into support of must-use values
* overwriting the value with ptr::write/copy
    * no solution, unsafe code
* aborting the program
    * no solution, programs die
* never leaving the scope the value is defined for
    * rustc's definite-init understands infinite loops and can error in many cases, but not all
    * e.g. `loop { do_it() }` can be found, but not `while never_false() { do_it() }`
    * erroring out on any potentially-infinite loop sounds really bad though?

Panicking is however a major issue for the checker. Any panic can lead to the program continuing, but without an opportunity to process the value. I see two options:

1) Introduce a no_panic effect/property/trait that functions can have. Only let no_panic functions be called while a must-use value is in scope.

2) Give all must-use types a destructor bomb that aborts the program. Since all must-use types are eventually passed to mem::forget, the only way a must-use type's destructor can run is if unwinding tries to destroy it.

Solution 1, like all effect systems, is a huge infectious burden. Solution 2 is a bit depressing but probably the right choice to make this even vaguely usable. Anyone who opts into a panic=abort build won't have to worry about this. 





## The Trait

First off, we have the issue of legacy defaults: all generic code today implicitly doesn't support must-useness. That's probably for the best, as generic must-use code is pretty awkward to write. [^swift-copy] Thankfully, Rust has already faced this issue once before in the form of Sized, and developed a solution for it. 

All generic types in Rust are implicitly Sized, but generic code may opt out of this assumption with the `T: ?Sized` bound. Here "?" is read as "maybe". So rather than being definitely Sized, we are now saying that something is maybe Sized, which is exactly what the absence of any other trait bound means: the type specified by `<T>` is *maybe* Copy.

I suggest introducing a new trait called Leave, which Drop extends. This is in analogy to Copy and Clone. Leave basically states "this type can be left unused". If a type is Leave but not Drop, then dropping the value is a no-op (`intrinsics::needs_drop::<T>() == false`). All types implicitly derive Leave. All generic code has an implicit Leave bound. This bound can be opted out of by adding `T: ?Leave`. 

A type that is definitely not Leave is said to be must-use. Any type that contains a must-use type must also be must-use. Generic types will derive must-use in much the same way as they do move-only:

```rust
// unconditionally must-use              // unconditionally move-only
struct MyType1<T: ?Leave>(T);            struct MyType2<T>(T);

// must-use iff T: must-use              // move-only iff T: move-only
#[derive(Leave)]                         #[derive(Copy, Clone)]
struct MyType3<T: ?Leave>(T);            struct MyType4<T>(T);
```

This doesn't seem that complex of a language extension. The two big downsides are:

* It will pollute up all sorts of APIs which Now Feature ?Leave Support
* It will lead to pain whenever you run into an API which Doesn't Yet Support ?Leave (because it depends on other APIs which don't, which in turn depend on...)

Having trouble thinking of exactly the right way to encode "a function that must be called at least once" for functions that capture must-use values. The negative bound of ?Leave combined with the backwards thinking of Fn traits hurts my brain right now.




## Usage Examples

A bunch of the standard library will need to be "upgraded" to have ?Leave support. Here's some pain points I see.

Let's look at Option:

```rust
#[derive(Leave)]
pub enum Option<T: ?Leave> {
    Some(T),
    None,
}

impl<T: ?Leave> Option<T> { ... }
```

Ok that works fine. Let's try to use it...

```rust
let mut token = None;
while cond1 {
    if cond2 {
        token = Some(step1()); // ERROR 1: assignment to already initialized must-use value
    }

    if cond3 {
        step2(token.take().unwrap());
    }
}
// ERROR 2: must-use value must be used
```

In both of these cases, the compiler is correctly identifying places where we might be accidentally discarding a must-use value. Too bad if you're certain the value is None at these places! The general solution is to `map` over the value and process it there. In our case we always expect it to be None, so we just abort if that isn't true:

```
token.map(|x| abort("token should have been None"))
token = Some(val); // OK!
```

This will likely be a common enough idiom that we provide some special method that does this:

```rust
let mut token = None;
while cond1 {
    if cond2 {
        token.assert_none();
        token = Some(step1());
    }

    if cond3 {
        step2(token.take().unwrap());
    }
}
token.assert_none();
```

Collections like Vec and HashMap are just a generalization of the Option situation, but instead of asserting is_none(), you have to assert is_empty(), and instead of calling `map` you call `into_iter`. 

But, collections have many subtle things that wouldn't work anymore.

* `array[i] = val` (old val not returned)
* map.insert/remove (old K not returned)
* truncate, resize, clear, dedup, retain, filter, ...

Some of these would be solved by dependent types, but that's like the most catastrophically complex language extension there is. Others like insert, remove, and retain would need replacement APIs that return all the values that might be replaced. This is a lot of API work. Basically a full audit of the standard lib, with lots of new API design work.

Actually, now that I mention it, I'm not actually certain how into_iter is supposed to work? The for loop consumes the iterator, but what if it held must-use data that isn't part of the yield? I think the loop just has to mem::forget all iterators passed to it unconditionally, and if you did something weird, that's your problem? Ick.





# Conclusion

Ok having actually written this all out, it's better than I thought. That said, I still think it's *a lot* of paper cuts that pile up to make an overall unpleasant and difficult to implement system. The library work is probably the biggest burden. Something that would realistically take years to shake out.

Feel free to call me a fool, or ask me questions, on twitter or any of the usual places The Rust Evangelism Strikeforce hangs out.



[^drop]: Technically it calls `drop_in_place()`, which calls `drop()` and then recursively calls `drop_in_place()` on all fields.

[^pass-by]: There's a bunch of hairy details about moving subfields and stuff I don't care about getting into here.

[^swift-copy]: The Swift devs have basically the exact same argument for move-only code, and their implicit Copy bound. Hooray!

