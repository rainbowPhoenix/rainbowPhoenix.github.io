---
layout: page
title: Inaugural State of the Site Address
short_title: About
show: true
---

Update 1 - What is a game?

(Note: This post assumes a passing familiarity with Finite State Automata. If

(Note: All the code from this post is on github here: )

In my last post, I outlined a project I called Delectibles (A contraction of 'Digital Collectibles') that would allow users to play turn-based games that used digital collections by allowing the users to verify each other's collections and then used Mental Poker to prevent cheating (cheating being either the acquisition of hidden information or the ability to manipulate hidden information). But this was clearly several independant codebases. Permitting a user to bring a game piece into the game by verifying that they actually own said piece is a completely separate problem than preventing cheating by manipulating pieces that are already in the game. And Mental Poker can authenticate actions taken by users, but it says nothing at all about whether those actions are consistant within the rules of the game that is being played. So what was proposed as one monolithic library is actually three: one to prove ownership of game pieces, one to prevent cheating, and a third to actually lay out the rules of the game.

This third piece is this focus of this update.

Our goal is to create a platform that can be used to implement generic turn-based games. Different games would be represented by different data files or scripts that the platform would use to lay out the game's rules. One option would be for the entire game to be implemented in these files, with an API that it uses to receive player actions from the rest of the application and sent the result of these actions. But would make it harder for us to prevent players tampering with the game. If a copy of the game is replicated on the computers of each user, we need to guarentee that nondeterministic behavior is also replicated faithfully for each client. So instead, we need to either require games be written a scripting language interpreted by our application, so that any nondeterministic behavior or hidden information is properly handled, or we provide libraries to handle this problem behavior and require the games to call into these libraries instead of handling it themselves.

I chose the second approach for now, because it would give people trying to implement their game on this platform more flexibility to use their own tools and languages, would require less work for them to port an existing game to my platform, and would require less work on my part, since I can use an existing compiler or interpreter instead of having to design one. Libraries I provide could also provide support for structures common to board games, such as shuffled decks or gridded boards.

For this proof-of-concept, I decided that the game files (and the rest of the implementation) would be Coffeescript scripts, a language that compiles down to Javascript. I chose this so that any code written here could be easily tested, modified, and experimentedon  by readers on whatever device they are using without needing to install anything.

Ideally, we want this platform to be able to support as wide a variety of games as possible. When designing an API for this library that game scripts will call into, we don't want to rule out a specific game or type of game because the library provides no way to receive necessary input or convey necessary information to the player. This requires us to define "game" in an unambiguous, mathematical sense.

So, what is a game? We want a definition that can describe as many different games as possible, for flexibility.

First off, a game must present players with choices. This is the key quality that makes something a game or not. If you're not influencing the game's outcome in some way, you're not playing, you're just watching. These choices are made based on information made available to the players. We call this available information at a specific point in time a **game state**.

The game state is the sum of all information available to any player about the state of any game piece and the value of any other variables that could influence decision-making (such as players' scores.) It's a flexible definition, but it must be in order to accomodate many different games. For any game, there are a (potentially infinite, but enumerable) number of game states.

One necessary component of the game state is the **active player**, the player who must make the next decision. Informally, this is the player whose turn it is. There are some games that require players to make decisions simultaniously, but this can be simulated through the use of cryptographic committment schemes (which I will discuss in a later post, when we actually start implementing cryptographic features).

While a game is being played, it moves through a sequence of different game states, with the exact sequence determined by choices that the players make, (and in some games, by chance). For any given game state, there are several choices the active player can make, each of which will cause the game to transition to a new game state. Each of these choices, or **options**, can be formally represented as a pair (s, t), where s is the game state before the option is chosen, and t is a (possibly nondeterministic) function that transforms the game state s.

We can formally represent a game as a tuple (S, →), where S is the set of all possible game states, and → is the set of all options. In this way, a game becomes a state machine. (A formal state machine requires that each state transition be a pair of an input and output state. We can achieve this by saying that if any option is nondeterministic, then each possible resulting state has its own edge in the state machine.)

This notation allows us to completely describe any game in a table, merely by listing the possible game states and listing all transitions. This doesn't mean that such a table will necessarily be short (or even have finite length!) For example, the game of Tic-Tac-Toe has 5,478 possible board states and 549,946 different possible options. (http://www.mathrec.org/old/2002jan/solutions.html) A more complicated game like chess could still be reperesented in this manner, but chess has 10^47 possible board states, and listing them all would be extremely impractical.

What is practical is calculating the part of the table that is immediately relevant for a given game state. Let's say that we have a tic-tac-toe board in the following position:

