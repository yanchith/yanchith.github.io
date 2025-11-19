I used JAI this summer for a videogame project (https://withoutfearofwindorvertigo.com/prototype)
that ended up being between around 15k lines of code at the time of writing, so not huge, but not
tiny anymore. The code constitutes a 2D puzzle game, running both natively and in the browser, and
an automatic solver for the game's puzzles. If the game grows, there will likely also be a puzzle
generator, more powerful asset system, better graphics, etc.

The overall experience was that I didn't really have to think about the language very much. It let
me to do what I wanted, leaving the responsibility of cleaning my own mess to me, but also giving me
the tools to do the cleaning. I prefer it this way, because, on the one hand, games need a lot of
iteration and trying things out, but on the other, finishing a game requires a fair amount of
hardening: systematizing, fixing bugs, optimization. So, at the beginning of a project, you want to
be messy and unconstrained, and by the end you need to be organized and precise. JAI allows you to
tune this dial as needed, and gives you tools to do so (metaprogramming, profiling tools,
instrumenting allocations, visualization tools like codex view, etc). To be fair, in this project I
utilized the "let me make a mess" aspect more than the "help me clean it up" one, both because the
game is small enough for it to not require large-scale hardening and this also isn't my first game,
so at this point I can avoid some programming and design mistakes ahead of time.

JAI is very much a language where you have to know what you are doing. Coming from a more
restrictive environment - until this summer, a C-like subset of Rust has been my programming
workhorse, for which I have a great deal of respect, but with thich I have also accumulated a fair
share of grievances - I found it refreshing to be able to just do things without having to prove to
the compiler they are correct. I apologize in advance to people with hardcode C background: some of
the ideas presented here will likely be obvious to you, but they were not immediately obvious to me
a few years ago, and I had to grow into them.

Here's the things I liked, followed by a few I have reservations about. Items in both lists came to
be by what I encountered while working on the project, as well as what just happened to be in my
head at the time, so they are by no means complete.

A quick list of things I really enjoyed:

- Pointers and indices are 64-bit and signed. No size_t/usize that have different sizes based on the
  target CPU. Being signed, you don't need to care about underflow when doing arithmetic. This is a
  breath of fresh air. Yes, the always 64-bit thing makes it hard to port JAI to non-64-bit
  platforms, so it is something to consider, if you know you are going to need that. However, all of
  PC, servers, Mac, the consoles, phones and tablets, and now even WASM is 64-bit, so you do get
  quite a bit of platform support even with this limitation.

- Integer types widen automatically, but truncating or reinterpretting requires an explicit
  cast. Perhaps it is obvious that this is how it should be, but both C, C++ and Rust do get this
  wrong. C and C++ have implicit conversions even in the wrong direction, while Rust goes overboard
  and requires you to explicitly convert a smaller integer type to a larger integer type (but at
  least you can do it with the From trait, so you can tune your house style so that you are not
  alerted by the cast).

- Structs are plain old data and can contain any bytes you like. Again, this is obvious, and yet
  people get this wrong. In JAI, there no such thing as compiler-exploited padding (or other unused
  bits) that would escalate your correctness bugs to UB. In languages with heavy typesystems, there
  is a "correct by construction" philosophy: you can only create correct values for a type
  (e.g. NonZeroU32 in Rust). You can imagine creating your own types whose constructors would
  enforce various invariants. While it initially might seem enticing, I disagree with this
  philosophy completely, as the cost far outweights any benefits. The cost is that you can not treat
  these values as plain data anymore. Zero-copy deserialization is now not possible, because the
  compiler could have generated code that acted data stored in the sturct padding, and just the act
  of creating a value with an invalid bit pattern is instant UB. Need to deserialize gigabytes of
  data from disk? If it has attached invariants that the compiler tracks, you can not just
  reinterpret the memory after you load the file, but also have to validate before you cast,
  otherwise you'll get UB. Fortunately, in JAI, you get none of this.

- There is a standard allocator interface. There are not many things that require the language to
  define a standard interface, but allocators are one such thing. This is a language where it should
  be easy for you to swap allocators per function, module or datastructure.

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
  - enum_flags, yes thank you.

- how_to files and modules that ship with the compiler are good documentation. The former is not
  just a tutorial, but spends a lot of time explaining reasoning and philosophy behind the
  language. Being able to read (and modify) the latter gives you the sense of being in control over
  the project. If you find a bug, you can just fix it for yourself locally and not have to wait
  while the maintainers respond to your report or bugfix.

- There's actually a lot of fancy stuff: metaprograming (The Compiler module, #run, #insert,
  #no_reset, #caller_location), autobakes, macros, etc, that I didn't get to use very extensively yet,
  because I was programming a small game that doesn't need all that power. I can imagine at
  least the metaprogramming stuff is tremendously useful for building stuff like profilers or code
  analyzers.

- Very fast compile times, unless you invoke the LLVM backend (for optimization or platform
  support), in which case they were still be quite okay.

I also have a few things I have a few reservations about:

- This is on the roadmap, but currently it is only possible to do SIMD for x64 (with the built-in
  inline assembler for X64). There's multiple possible approaches for how Jon and the team can
  decide to handle this - some kind of general SIMD abstraction (this already has an API prototype
  in modules/SIMD.jai) vs. platform intrinsics vs. userspace inline assembler - so it might
  take some time before SIMD is possible to do without friction on all platforms.

- Context is convenient, but I don't think I got a lot of mileage out of it, and I had to pay for it
  (slighly). I only use the context allocator, temporary storage and context logger, and can't think
  of anything else I'd like to put there. For the allocator, I could have lived with, or even
  preferred passing the memory arenas as explicit parameters compared to setting the context
  allocator. For logging, the way I use a logger is amenable for it being just a global
  variable. When another language calls a JAI dynamic library, you have to create an empty context
  in all entrypoints, which is inconvenient. I am considering that for future projects in the
  language, I won't use Context in my programs, perhaps with the exception of overriding the
  allocator of libraries someone else wrote.

- The #poke_name thing. Modules by default do not see code from other modules. This is sometimes
  inconvenient, one example being when you make a struct and you want to use it as a parameter to
  the array_find function, or a key in a hash table. By itself, the callee module doesn't see the
  operator== for the struct. #poke_name is the temporary workaround that lets you make the module
  aware of your stuff, but it is not very ergonomic or transparent. I am looking forward to having a
  better version of this.

- I personally would like a stricter separation of OS-specific from platform-independent code. These
  days, I default to having explicit, Casey-style platform layers, but the JAI modules go the (far
  more travelled) route of abstracting over the common features of multiple APIs in their
  internals. I can still do the Casey thing, but for that I have to ignore some modules shipped with
  the compiler (File, Thread, Window_Creation, Input, Simp, Sound_Player) and make my own. However,
  there's a few things I can't really change, like Temporary_Storage being partially defined in the
  compiler, or write_string being defined by Runtime_Support.jai. Overall these ideals of mine are
  quite very platonic, and in reality I don't mind the more pragmatic approach that Jon took.

- I get why unused variables are not warned against, and that it can be annoying, but sometimes an
  unused variable warning points to an underlying problem even when rapidly iterating on code
  (e.g. when incorrectly shadowing). Maybe it can be optional and disabled by default, or maybe a
  metaprogram plugin?

Since I am mentioning my reservations, now would be the time to also mention the bus factor. The
team working on the compiler at Thekla is very small. I wouldn't start a large project with JAI just
yet, unless my team gets source access to the compiler and there is budget in the project for
maintaining JAI.

Otherwise, for small projects, JAI is already a great boon, and I am looking forward to having the
polished and fully released version of the language that I can use for large projects.
