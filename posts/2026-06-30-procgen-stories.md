# Procedural Story Generation and Storylets

*Published: 2026-06-30*

During one of the game jams I've explored the idea of simple journal generation that would later release as [Signal Lost](https://vuvko.itch.io/signal-lost) jam entry.
My main touchstone for the jam was the [Godville](https://en.wikipedia.org/wiki/Godville) — a zero-player game, where the player watches after the mythic hero with very limited influence.

After that, I've looked into procedural puzzle generation that I tried to incorporate into my [7DRL](https://7drl.com/) entry [Coherence](https://vuvko.itch.io/coherence-7drl).
It was a simple lock-and-key structure with some predefined rooms (prefabs) to add complexity.

But both those projects lacked depth and any emergent stories you can have in games like [Dwarf Fortress](https://en.wikipedia.org/wiki/Dwarf_Fortress), [Wildermyth](https://en.wikipedia.org/wiki/Wildermyth), and others.
And the answer was not as simple as more prefabs and more branches.
First, managing such a big database of templates and text parts is almost impossible.
Second, the depth would only grow linearly on number of branches. 
Another way to structure the storytelling was needed.

## The Limits of Templates

To understand why the template expansion would not work, the best example would be to try and do that.
In Coherence on every level the player need to find the terminal with an extraction key in order to proceed.
On lower levels, it is a simple task of finding the room, then proceeding to the exit, while dodging enemies.
To make it harder on middle levels, some of the rooms preceding the terminal room are hazardous and require the player to either push through them or find a deactivation terminal for them.
This adds an additional level of complexity.
On the last level the terminal room is always under quarantine, thus making a three-level puzzle of finding the right key before proceeding.
This, however, quickly runs out of interesting branches on exploration.
As the player would proceed with mindlessly searching for all terminals and turning off every hazardous room until completion.
There is no additional depth emerging from having stacking layers of simple repetitive puzzles.

In Signal Lost I experimented with LLM-as-a-Judge pipeline to scale the generation and management of templates in hopes to tackle the problem.
It was a poorly-designed pipeline but it succeeded in generating additional lines the autonomous agents were adding to their journals.
But, it did not add to the fun on diversity of the journals as can be seen in some of the [reviews](https://itch.io/jam/ai-browser-game-jam/rate/4328367) for the game.

## Managing the Chaos

Every narrative-based game uses an engine that helps to store the narrative part in an easy-to-read way.
[Ink](https://www.inklestudios.com/ink/), [Twine](https://twinery.org/), and similar store as branched dialogues with special markers.
They are easy to read and manage with the most of the storyline being presented as a linear set of scenes (there are a lot of ways to use these tools in a much different way).
Other engines use a special database to store narrative parts of a game. There are also academic publications on procedurally generating narrative puzzles[^puzzles] and generating grammars for missions[^adventures].

### Storylets

But recently, I've encountered a new structure for me — storylets[^storylets].
Those are parts of the story (can be as small as one sentence) that have prerequisites to fire in-game, and have an impact on the state of the game.
This is similar to event-based programming or pattern-matching guarded commands.
And it should help in both generating and storing them in a database.

With the key advantage that you can separate the story parts from the overall structure, you can work with independent or small-scale parts without the worry about retaining the whole branching tree structure.
You can also much more easily create new ones that would make sense in the context of prerequisites.
And the depth should emerge with the growing number of potential interactions between different parts.
That would be problematic to track via a simple branching tree.

### Other works

I also found plenty of academic research in the adjacent space: grammar-based generation[^adventures], storylet architectures[^wawlt][^drama_llama], and story recognition systems that query emergent narrative for meaning[^felt] — with tests on users to determine what is working.

But there is another ground for inspiration that is dear to me, and that a lot of people try to digitalize for a long time — tabletop games.

## Tabletop Roleplaying Games

I briefly describe the tabletop experience in one of the previous [posts](./2026-06-25-pfs-return.md).
And it has a lot to do with how stories can be procedurally generated.

It is hard to improvise at the table without any clue or hint.
This is why a lot of game systems provide a variety of elements to help structure the scene.
There are established facts you can always refer, or establish your own facts (with a good roll in [Fate](https://en.wikipedia.org/wiki/Fate_(role-playing_game_system)) or [Powered by the Apocalypse](https://en.wikipedia.org/wiki/Powered_by_the_Apocalypse), preparation steps in [Fiasco](https://en.wikipedia.org/wiki/Fiasco_(role-playing_game))).

TTRPGs also have an almost universal approach to generating interesting stories where the players state the scene, but the outcome is determined by dice rolls.
It is interesting in the procedural generation because you can rationalize the results after the established fact of completing or failing an action.

## Synthesis

The storylets structure is open enough to design any story or narrative tree.
Following the TTRPGs flow, we can add more restrains on pivotal parts of the story that would be resolved with dice:

* The actor should have agency in it.
* They should have a potential consequence to the actor upon failure.
* The result is determined by actor's skills and its position.

We can also add partial successes and failure from more narrative TTRPGs so that the outcome of the action could be more dramatic than a mere "miss".

I intend to experiment with this idea further in an update to my Signal Lost game as further research into this topic.

---

> This post is part of my [#100DaysToOffload](https://100daystooffload.com/) challenge.


[^puzzles]: Fernández-Vara, Clara, and Alec Thomson. ["Procedural generation of narrative puzzles in adventure games: The puzzle-dice system."](https://dl.acm.org/doi/10.1145/2538528.2538538) Proceedings of the The third workshop on Procedural Content Generation in Games. 2012.

[^adventures]: Dormans, Joris. ["Adventures in level design: generating missions and spaces for action adventure games."](https://www.pcgworkshop.com/archive/dormans2010adventures.pdf) Proceedings of the 2010 workshop on procedural content generation in games. 2010.

[^felt]: Kreminski, Max, Melanie Dickinson, and Noah Wardrip-Fruin. ["Felt: a simple story sifter."](https://mkremins.github.io/publications/Felt_SimpleStorySifter.pdf) International Conference on Interactive Digital Storytelling. Cham: Springer International Publishing, 2019.

[^wawlt]: Kreminski, Max, et al. ["Why Are We Like This?: The AI architecture of a co-creative storytelling game."](https://dl.acm.org/doi/10.1145/3402942.3402953) Proceedings of the 15th International Conference on the Foundations of Digital Games. 2020.

[^storylets]: Emily Short. ["Storylets: You Want Them."](https://emshort.blog/2019/11/29/storylets-you-want-them/) (2019)

[^drama_llama]: Sun, Yuqian, et al. ["Drama llama: An llm-powered storylets framework for authorable responsiveness in interactive narrative."](https://arxiv.org/abs/2501.09099) arXiv preprint arXiv:2501.09099 (2025).
