In between 2021 and 2024, I was working on a game. The game's core idea as it ended up (it evolved
during development) is something that inspires me even today, and I would still like to make a
similar game at some point, but at the time I was working on it, it was way too ambitious of a
project for my skill level.

The game is a 3D exploration sokoban with realtime movement, where the player controls multiple
characters - party of adventurers. There is an overworld, where most of the exploration happens, and
there are levels accessible from the overworld, where more focused puzzles are solved. Solving
puzzles unlocks new ways to progress on the overworld, either becuase the unsolved level literally
was blocking the path forward, or by knowledge gating: the player would understand an emergent
property of a game mechanic inside a focused level, and could then use that property to get to
previously unvisited parts of the overworld.

While it wasn't set in stone, when I thought about the mood and feel of the game, it was a
Zelda-esque thing, where the party of adventurers advanced through an fantasy-themed and somewhat
hazardous world to... find treasure? rescue a princess? slat a demon and save the world? I never
really decided. This would have become clear as the game neared shipping. No story (or an obscure
and hidden story) was also a possibility.

In this post, or perhaps a series of posts, I'd like to talk about what I did. Most of it is
probably going to come off as very negative, but I did look at the game recently, and I honestly
also felt joy and pride about how much stuff I've built.


# building an engine
- allocation, memory management, collections
- portability
- serialization and versioning
- entity systems
- rendering

# Designing a game

- 3D sokoban
- realtime (asynchronously moving entities)
- wrapping coordinates
- multiple characters
- multiple characters controlled at once
- overworld and levels

- designing mechanics and levels
- which characters are on the overworld? party character vs individual

# pain points

2D -> isometric -> 3D

Experimentation was expensive.

Experimentation was at first slow, because serialization/migration code had to be modified each time.

Experimentation was slow, because even at the core of the game, many mechanics already
interacted. For example, the code that managed movement was about a 1000 lines, and doing any
mechanic that interacted with movement meant careful re-examining and editing of this code
(sometimes just small parts of it, but sometimes the whole system had to be rewritten to support the
new mechanic)
