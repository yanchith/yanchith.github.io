I used JAI this summer for a videogame project that ended up being between around 15000 lines of
code at the time of writing (XXX: Link), so not exactly a tiny toy program anymore.

The overall experience was that of being productive and the language allowing me to do what I want,
leaving the responsibility of cleaning my own mess to me. I prefer it this way, becuse, on the one
hand, games need a lot of iteration and trying things out, but on the other, finishing a game
requires a fair amount of hardening: systematizing, fixing bugs, optimization. So, at the beginning
of a project, you want to be messy and unconstrained, and by the end you need to be organized and
rigid. JAI allows you to tune this dial as needed, and gives you tools to do so (metaprogramming,
profiling tools, instrumenting allocations, visualization tools like codex view, etc). It is very
much a language where you have to know what you are doing, and it gives you ways to shoot yourself
in the foot, but it also doesn't restrict you unnecessarily. Coming from a more restrictive
environment, I found it empowering to be able to just do things without having to prove to the
compiler they are correct.

Here's a few things I liked, followed by a few I have reservations about. My perspective is that of
a person that cares about engineering. My most used language until this point has been (a C-like
subset of) Rust.

A quick list of things I really enjoyed:

- Pointers and indices are 64-bit and signed. No size_t or ptrdiff_t that have different sizes based
  on the target CPU. Coming from Rust, which has you caring about "could this usize potentially be
  16-bit" everywhere, this is a breath of fresh air. Yes, this makes it hard to port JAI to
  non-64-bit platforms, so it is something to consider, if you know you are going to need that. On
  the other hand, all of PC, servers, Mac, the consoles and now even WASM is 64-bit, so you do get
  quite a bit of platform support this way.

- Integer types widen automatically. Truncating requires a cast. As it should be.

- "#as using base: Base_Struct;" let's you quickly implement shallow inheritance hierachies. You can
  store each type in separate storage, so you don't pay the cost for the largest type everywhere,
  but by passing a pointer, "#as using" lets you treat data homogenously when you only want to look
  at the common fields. Yes, you can do this in other languages (inheritance in C++, "the Deref
  trick" in Rust), but it is especially easy to do in JAI. One thing to note is that JAI has neither
  a massive type system nor inheritance, but captures the useful feature as its own thing. Well, two
  things - #as and using.

- Overall syntax decisions and terseness. I don't usually talk about syntax, but a few things I like:
  - $T for declaring and using a polymorphic parameter at the same time
  - Skipping type name for enum and struct init syntax: "x = .MY_ENUM_VALUE;"
  - Very terse loop syntax: for 0..10, for array, it, it_index.

- how_to files and modules that ship with the compiler are very good documentation. The former is
  not just a tutorial, but spends a lot of time explaining reasoning and philosophy behind the
  language. Being able to read (and modify) the latter gives you the sense of being in control over
  the project. If you find a bug, you can just fix it for yourself locally and not have to wait
  while the maintainers respond to your report or bugfix.

- There's actually a lot of fancy stuff: metaprograming (#run, #insert, #no_reset,
  #caller_location), autobakes, trashy closures with #expand macros, etc, that I didn't get to use
  very extensively yet, because these are more suited when making a library, and I was programming a
  game. I can imagine at least the metaprogramming stuff is tremendously useful for building stuff
  like profilers or code analyzers.

I also have a few things I have a few reservations about, mostly because I tend to be more careful
about how the code is organized. Most of these have a simple solution: don't use that feature.

- Context is very convenient, but it allows for a fast and loose style that I have to force myself
  to use. This may be my strict upbringing (A combination of Rust and being a fan or arenas), but
  each time I use context to modify the allocator, I'd rather have explicitly passed that arena
  myself. The only other thing I use context for is logging, and I am much more okay with that,
  maybe becuase it is much less volatile - logging is usually set up once in the programs I write.
  I am considering that if I stick with the language, I might stop using Context in my programs. Or
  perhaps, I will yet find some use for it. Some other people in the beta say that it is 90% useful
  for overriding the allocator a library they didn't write uses.

- I personally would like stricter separation of OS-specific from platform-independent code. I like
  having explicit, Casey-style platform layers. I can do this, if I ignore some modules
  shipped with the compiler (File, Thread, Simp, Sound_Player) and make my own.

- I get why unused variables are not warned against, and that it can be annoying, but sometimes an
  unused variable warning points to an underlying problem, even when rapidly iterating on code
  (e.g. when incorrectly shadowing). Maybe it can be optional and disabled by default? Maybe it
  already exists as a metaprogram plugin?
