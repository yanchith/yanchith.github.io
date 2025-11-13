In summer of 2025, I found myself with some free time on my hands, and at the same time creatively
unfulfilled. (These are related, but that is a different story.)

I had some money saved up, giving me a few months of runway to make something before having to look
for a new source of income. I have also been playing a lot of Dragonsweeper (XXX: Link) and Dungeons
and Diagrams (XXX: Link) in the preceding months, so my head was full of ideas about deduction and
observation gameplay. On top of that, I have had access to the JAI language beta for quite some
time, and felt bad that I still haven't done a nontrivial thing with it yet.

And thus, a naive motivation came to be: make a game that is like Dungeons and Diagrams, but has
Dragonsweeper observation gameplay. Also, because this is going to be just a simple 2D game this
time (right?!), it is fine to program it in a language I am not proficient in, so that I can form an
informed opinion.

The JAI part probably deserves its own small post, but I'll constrain it to a few sentences. The
language is good! It has rough spots and could use polish in places, and I was missing my libraries
from previous projects, but I was almost instanly more productive with it than with my previous
workhorse. I believe this is because a deep design principle of JAI is to trust the programmer,
which then surfaces in many of the language's features.

On the one hand, games need a lot of iteration and trying things out, but on the other hand,
finishing a game requires a fair amount of hardening: systematizing, fixing bugs, optimization. So,
at the beginning of a project, you want to be messy and unconstrained, and by the end you need to be
organized and rigid. JAI allows you to tune this dial as needed, and gives you tools to do so
(metaprogramming, profiling tools, instrumenting allocations, codex view, etc) (XXX: Improve this
list). It is very much a language where you have to know what you are doing, and it probably gives
you many ways to shoot yourself in the foot, but it also doesn't do unnecessary restrictions and
guard rails. Coming from a more restrictive unvironment, I found it empowering to be able to just do
things without having to prove to the compiler they are correct.

Now on to the game.
