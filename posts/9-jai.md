I used JAI this summer for a videogame project (https://withoutfearofwindorvertigo.com/prototype)
that ended up being between around 15k lines of code at the time of writing, so not huge, but not
tiny anymore. The code constitutes a 2D puzzle game, running both natively and in the browser, and
an automatic solver for the game's puzzles. If the game grows, there will likely also be a puzzle
generator, more powerful asset system, better graphics, etc.

The overall experience was that I didn't really need think about the language very much. It let me
to do what I wanted, leaving the responsibility of cleaning my own mess to me, but also giving me
the tools to do the cleaning. I prefer it this way, because, on the one hand, games need a lot of
iteration and trying things out, but on the other, finishing a game requires a fair amount of
hardening: systematizing, fixing bugs, optimization. So, at the beginning of a project, you want to
be messy and unconstrained, and by the end you need to be organized and precise. JAI allows you to
tune this dial as needed, and gives you tools to do so (metaprogramming, profiling tools,
instrumenting allocations, visualization tools like codex view, etc). To be fair, in this project I
experienced the "let me make a mess" aspect more than the "help me clean it up" one, because the
game is still small enough for it to not require large-scale hardening.

JAI is very much a language where you have to know what you are doing. Coming from a more
restrictive environment (until this point, Rust has been my programming workhorse, with which I have
accumulated my fair share of disagreements), I found it refreshing to be able to just do things
without having to prove to the compiler they are correct.

Here's the things I liked, followed by a few I have reservations about. Items in both lists came to
be by what I encountered while working on the project, as well as what just happened to be in my
head at the time, so they are by no means complete.

A quick list of things I really enjoyed:

- Pointers and indices are 64-bit and signed. No size_t/usize that have different sizes based on the
  target CPU. Being signed, you don't need to care about underflow when doing pointer or index
  arithmetic. This is a breath of fresh air. Yes, this makes it hard to port JAI to non-64-bit
  platforms, so it is something to consider, if you know you are going to need that. On the other
  hand, all of PC, servers, Mac, the consoles and now even WASM is 64-bit, so you do get quite a bit
  of platform support even with this limitation.

- Integer types widen automatically. Truncating requires a cast. As it should be.

- "#as using base: Base_Struct;" let's you quickly implement shallow inheritance hierachies. You can
  store each type in separate storage, so you don't pay the cost for the largest type everywhere,
  but by passing a pointer, "#as using" lets you treat data homogenously when you only want to look
  at the common fields. Yes, you can do this in other languages (inheritance in C++, "the Deref
  trick" in Rust), but it is especially easy to do in JAI. One thing to note is that JAI has neither
  a massive type system nor inheritance, but captures the useful feature as its own small set of
  features.

- Overall syntax decisions and terseness. I don't usually talk about syntax, but here's a few things to call out:
  - $T for declaring and using a polymorphic parameter at the same time,
  - Skipping type name for enum and struct init syntax: "x = .MY_ENUM_VALUE;",
  - Very terse loop syntax: for 0..10, for array, it, it_index.

- how_to files and modules that ship with the compiler are good documentation. The former is not
  just a tutorial, but spends a lot of time explaining reasoning and philosophy behind the
  language. Being able to read (and modify) the latter gives you the sense of being in control over
  the project. If you find a bug, you can just fix it for yourself locally and not have to wait
  while the maintainers respond to your report or bugfix.

- There's actually a lot of fancy stuff: metaprograming (The Compiler module, #run, #insert,
  #no_reset, #caller_location), autobakes, macros, etc, that I didn't get to use very extensively yet,
  because these are more suited when making a library, and I was programming a game. I can imagine at
  least the metaprogramming stuff is tremendously useful for building stuff like profilers or code
  analyzers.

- Sub-second compile times, unless you invoke the LLVM backend (for optimization or platform
  support), in which case they were still less than 2 seconds for this project.

I also have a few things I have a few reservations about:

- This is on the roadmap, but currently it is only possible to do SIMD for x64 (with the built-in
  inline assembler for X64). There's multiple possible approaches for how Jon and the team can
  decide to handle this - some kind of general SIMD abstraction (this actually has an API prototype
  in modules/SIMD.jai) versus platform intrinsics versus userspace inline assembler - so it might
  take some time before SIMD is possible to do without friction on all platforms.

- Context is convenient, but it allows for a fast and loose style that I have to force myself to
  use. This may be just my habits (e.g. liking to explicitly decide which memory arena a piece of
  data goes in), but each time I use context to modify the allocator, I'd rather have explicitly
  passed that arena myself. The other thing people use context for is logging, and I am much more
  okay with that, maybe becuase it is less volatile, but on the other hand, the logger context could
  have been a global variable. I am considering that for future projects in the language, I won't
  use Context in my programs, perhaps with the exception of overriding the allocator of libraries
  that someone else wrote.

- The #poke_name thing. Modules by default do not see code from other modules. This sometimes comes
  up, one example being when you make a struct and you want to use it as a key in a hash table. By
  itself, the hash table doesn't see the operator== for the struct, nor its hash
  function. #poke_name is the temporary workaround that lets you make the module aware of your
  stuff, but it is not very ergonomic or transparent. I am looking forward to having this working in
  a systemic way.

- I personally would like stricter separation of OS-specific from platform-independent code. I like
  having explicit, Casey-style platform layers. I can do this, if I ignore some modules shipped with
  the compiler (File, Thread, Window_Creation, Input, Simp, Sound_Player) and make my own.

- I get why unused variables are not warned against, and that it can be annoying, but sometimes an
  unused variable warning points to an underlying problem, even when rapidly iterating on code
  (e.g. when incorrectly shadowing). Maybe it can be optional and disabled by default, or a
  metaprogram plugin?

Since I am mentioning my reservations, now would be the time to also mention the bus factor. The
team working on the compiler at Thekla is very small. I wouldn't start a large project with JAI
unless my team gets source access to the compiler and there is budget in the project for maintaining JAI.

Otherwise, for small projects, JAI is already a great boon, and I am looking forward to having the
polished and fully released version of the language that I can use for large projects.
