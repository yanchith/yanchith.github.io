In summer of 2025, I found myself with some free time on my hands, and at the same time creatively
unfulfilled. (These are related, but that is a different story.)

I had some money saved up, giving me a few months of runway to make something before having to look
for a new source of income. I have also been playing a lot of Dragonsweeper
(https://danielben.itch.io/dragonsweeper) and Dungeons and Diagrams
(https://steamcommunity.com/sharedfiles/filedetails/?id=2831798266) in the preceding months, so my
head was full of ideas about deduction and observation gameplay. On top of that, I have had access
to the JAI language beta for quite some time, and felt bad that I still haven't done a nontrivial
thing with it yet.

And thus, a naive motivation came to be: make a game that is like Dungeons and Diagrams, but also
has Dragonsweeper-like observation gameplay. Also, because this is going to be just a simple 2D
game, it is okay to program it in a language I am not proficient in, so that I can form an informed
opinion on that language.

While it might seem a little risky to both learn a new programming language and do a new project at
the same time, the programming went well, and I would change nothing about it. My experience with
JAI deserves a larger post, but I'll constrain myself here. I was almost instantly productive with
the language and it felt very good to program in! It has some incomplete or rough spots and could
use polish in places, but overall this did not matter very much. The productivity can be attributed
a multitude of smaller features (or sometimes a lack thereof) that play harmoniously together. I
believe this is because a deep design principle of JAI is to trust the programmer to keep things
orderly, and not insulate from the physical system (CPU, memory).

Now on to the game. The rest of the article assumes player knowledge of Dungeons and Diagrams.

The first iteration explored in the most straightforward way, what would it be like to add new,
unexplained mechanics to Dungeons and Diagrams. I prototyped three new mechanics, each represented
by a clue type in the dungeon. Two of these still exist in the second iteration - the one currently
released on the website.

One mechanic is traps. Traps have exactly two exits and two walls, and the exits and walls have to
be opposite each other. Traps have mildly interesting interactions with the rest of the game. The
not-so-interesting interaction is with all nearby clues, dungeon edges or placed walls, becuase one
fully specified tile (wall or hallway) adjacent to the trap forces the trap to a certain
configuration. The slighly more interesting interaction, and the reason why they are still in the
game is that they interact with the row and column numbers. Determining the trap adds two walls to
its row or column, but only one of these may be sometimes possible, which creates a deduction the
players can apply. The more advanced version of this deduction uses multiple clues in the same row
or column (e.g. two traps, or a trap and a monster) that, combined with the numbers make some
configurations either forced or impossible.

I am still not sure how I feel about traps. They were very simple to implement, so they are
relativelty low-cost, but the value also isn't that great. So far, I've only discovered one
interesting thing they do, and I'd feel way better about them, if they generated more than just
variations of the same deduction.

The second mechanic is mirrors. Mirrors are always rotated such that they faced diagonally
(e.g. North-West), and they have to see the same thing - clue or wall - on both sides. If a mirror
sees another mirror, they forme a mirror chain, which also has to see the exact same thing on both
sides. Sometimes a mirror chain could form a loop, in which case the mirrors just see themselves.

Mirrors did not survive to the second iteration, because while they look interesting on the surface
(and a similar concept has been used in other games to great success), they were not very actionable
to make deductions. At almost no time can the player solving the puzzle make deductions based on
what the mirror sees. They also had the tendency of hinting at the solution: for example, a mirror
seeing two treasure chests means there has to be empty space between the mirror and both treasures,
or there have to be two walls separating the treasures from the mirror, so they either almost spoil
the solution, or have almost no effect at all. Because they didn't generate more than I put in,
combined with the fact that the code validating mirrors was more involved compared to the traps, the
mirrors did not survive to the second interation.

The third mechanic is crests. Crests are either red or blue, have either one or two dots inside, and
always come in pairs. The number of dots is the number of distinct paths between the crests of the
same type. If there's two blue crests, both with one dot, then there has to be exacly one path
between them. If there's two blue crests with two dots, there have to be two nonintersecting paths
between them. The additional rule of crests is, that any path between crests is invalid, if it
intersects another path between a different pair of crests. This part sort of emerges automatically
between crests of the same color, because it violates the path count, but I was able to generate
more puzzles by forcing the inter-color interaction to also have this property.

Crests are complicated. The code that checks them is about one third of the entire puzzle checker
code. It also has to pull serious tricks to run within one frame's budget (or one day's, for that
matter), as the naive path counting algorithm is exponential in the size of the graph. On the
positive side: they do generate puzzles, some of which I think are the best puzzles I made in this
game. They are actionable in the sense that a well crafted crest puzzle requires the player to make
topological deductions to be solvable.

So these are the mechanics. At this point in development, I was still operating under the assumption
that the rules of the original Dungeons and Diagrams are explained explicitly by an animated
tutorial, and then the new puzzle clues have to be inferred from context by the players. I ran a
very small playtest, and one player managed to get to the ending of the game. However, I was
dissatisfied with how some things about how the game worked.

- the new mechanics felt wrong to teach by observation -> nonverbal tutorials for new mechanics

- second version
- second wave of playtests
