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

X| |X
- - -
 |X|O
- - -
 | |O

board = {X: [[0,0], [0,2], [1,1]], O: [[1,2], [2,2]]}
activePlayer = 'O'

We could create a table mapping each legal move to a transformation function and the resulting game state:

Choice   Transformation Function            New active player    New Game State
[0,1]        (board) -> board.O.push [0,1]    X                    {X: [[0,0], [0,2], [1,1]], O: [[1,2], [2,2], [0,1]]}
[1,0]        (board) -> board.O.push [1,0]    X                    {X: [[0,0], [0,2], [1,1]], O: [[1,2], [2,2], [0,2]]}
[2,0]        (board) -> board.O.push [2,0]    X                    {X: [[0,0], [0,2], [1,1]], O: [[1,2], [2,2], [1,2]]}
[2,1]        (board) -> board.O.push [2,1]    X                    {X: [[0,0], [0,2], [1,1]], O: [[1,2], [2,2], [2,2]]}

Viewing games as a group of generated tables like this suggests an API: Instead of listing every possible game state, a game descriptor file needs to be able to do one of the following:

Option 1: (Functional Programming Approach)
-For a given game state, enumerate the possible Options and assign each one a unique id.
-For a given game state and an option id, check whether the given option is possible and returns a new game state object that shows the

Alternatively the descriptor file can create and store a single game state, and:

Option 2: (Imperitive Programming Approach)
-For a given game state, enumerate the possible Options and assign each one a unique id.
-For a given game state, check whether the given option is possible and transforms applies the option to the game state, transforming it.

What’s so deceptive here is that at a first glance, it doesn’t even seem like there’s a choice to be made.  In my initial draft, the rules script created the functions and data structures that governed how the game was played, and created and stored a single initial game state, which would be transformed by submitted options. Instead of exporting an object that contained the entire game state, parts of the game state were only referenceable through function closures in the Option objects. This was for all practical purposes a strictly worse version of the second option. I came to realize that this was a violation of “separation of concerns”, the maxim that any unit of code (be it function or library or file) should do one thing and do it well. But my rules script was trying to do two things when really, it should lay down the rules and nothing more. It should describe how to create the initial game state, but should not actually do it.

This is perhaps the most important design decision in this update, even though it doesn’t immediately seem like one. In my experience, the most important decisions are usually the ones that don’t even look like decisions. Learning to recognize these hidden opportunities is an important skill for any software engineer.

So how do we proceed from here?

There are groups today that push strongly for a functional programming approach in everything. And it does have its benefits. For example, the first option has several advantages over my original naive implementation:

-Building test cases is much easier when the test case can supply an already built game state.
-The rules file only has to be loaded and processed once, and several games can be run simultaneously.
-The game state can be saved out to a file or data structure and loaded back in, allowing games to be paused and resumed, or a game to begin from a configuration other than its normal initial configuration.

But these features are also possible with option 2, but the first option still has some benefits over the proposed second:

-Making the game state object immutable means that a reference to the game state can be kept as a snapshot of the game at a particular point in time, without having to worry about the object being modified or having to create a deep copy of it. This is useful for recording purposes.
-A player can simulate future outcomes from a particular game state without having to worry about mutating the game state or creating copies of it. This is useful for AI or for a user interface that lets a player experiment with future possible outcomes.

But this method has several drawbacks, related to the process of creating a new game state from the existing one.

-If a new game state object is created every time an option would change it, this can lead to lots of unnecessary object creations and destructions, a performance penalty that scales with the complexity of the game state. I refuse to just assume that game states can always be easily created.
-Creating a new game state from an old one requires knowledge of how the game state is structured, and isn’t necessarily a trivial task.

These two drawbacks are wholly dependant on how the game state object is represented internally. If we can devise a representation that can easily be instantiated from an old game state and an applied Option, without adding restrictions or complications to the process of creating a rules file, then the drawbacks are inconsequential, and it becomes better to take the functional programming approach.

The key insight comes from recognizing that any game state is created by sequential **effects** to an initial game state, where an effect is the outcome of executing an option. It’s possible for us to represent the game state as this list of effects. If the list is a linked list, we can append new effects to the front of the list, creating the new game state in constant time. However, now every time part we want to read a value from the game state, we must iterate through the list until we find the relevant bit of information. We have traded a logarithmic construction time for a linear read time, and this is very likely a bad tradeoff. However, we can mitigate this by caching these reads in the most recent effect. Provided that a value in the game state is read with approximately regular frequency, the read will take constant time on average, and only values that are rarely read will take linear time. This seems like an acceptable tradeoff for the hassle of assembling game state objects of potentially dubious complexity (and possibly having to hand off some of this complexity to the rules file.)

The remaining concern is whether this internal representation can be hidden from the rules file. We will see how this is done later.

With all this research and planning behind us, let’s start writing code. We need to nail down an API to be used by the rules file to convey the rules for a game to the game engine. The best way to create a sensible and predictable API  is to create a use case first: let’s create a prototype rules file and see how we would instinctively want to express the rules for a simple game like Tic Tac Toe.

The rules file is currently a coffeescript script that is loaded and executed in a special environment. The major downside is that this requires that the rules file be trusted, so this is by no means a final solution.

What’s the most important part of a game? The players. So let’s tell the engine about the players.

``` coffescript
X = new Player(“X”)
O = new Player(“O”)
```

The string parameters are ids that we can use to reference the players later, since the variables X and O will only exist in this local scope.

Like all objects in coffeescript, X and O can have arbitrary properties defined on them. We can do that now:

``` coffeescript
X.opponent = O
O.opponent = X
```

There is no builtin way to dictate turn order, since not all games settle for something as simple as “clockwise around the table.” We will handle turn order by having each player pass the baton to the player listed as their opponent. (We’ll get to that in a bit.)

``` coffeescript
Game.setPlayers [X, O]
Game.setActivePlayer X
```

The goal is to create a game rules object that contains all the information necessary to govern a game. These two commands configure the rules object with a list of players and the starting player.  Instead of making these calls, the rules script could also return a dictionary with these values. (Both are perfectly acceptable.)





exports = exports ? this

global = (name, data) ->
    exports[name] = data
    
global "global", global

# Frankly, I feel this should be a builtin feature of coffeescript. It's simple enough to implement on my own (only being four short lines), but it's expected behavior.

#Some people propose using @ to make a global variable, but this only works because @ is a shorthand for "this." everywhere, and when you're not in a coffeescript function, this refers to the window.

I wanted to use “export” instead, but export is an outdated javascript builtin, and thus can’t be used as a variable name.

A note on the potential powers of prototypal inheritance.
