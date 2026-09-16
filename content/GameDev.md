---
title: Godot And GameDev
permalink: https://github.com/Pi3dra
---

I played a lot of videogames as a kid. After a few university projects in which I had to develop small games, I decided to bite the bullet and try out some **Game Jams** and develop my own games in a good game engine like [Godot](https://godotengine.org/).

So I teamed up with a cousin who also knows how to program and loves videogames, and started participating in game jams.

For those unfamiliar with **Game Jams**, these are community events, in which people try to develop a game, with a restricted theme, in a reduced amount of time. This can range from a weekend to a few months.

If you are interested in participating in game jams, you can find many in [here](https://itch.io/jams).

In these blog posts, I will be talking mostly about my experience, and my opinions on game design, and not really on how I coded these games.

At the time of writing, I have participated in 2 GameJams:


# Brackey's Game Jam 2025.2 

The theme was **Risk it for the biscuit**
It lasted 1 week, with 2,279 games submitted, and we finished with an overall **rank of 242**!

I would say that, compared to the [[#GMTK 2025]] game jam, this one was a really nice success, I learned even more, felt way more at ease with [Godot](https://godotengine.org/), and the game was really original.

You can **check it out** [HERE!](https://itch.io/jam/brackeys-14/rate/3851523) 

The game loop was heavily inspired by **auto-battler** games and current **card games**.
Here we didn't have actual cards, but cookies, which the player has to flip, and if they landed on the heads side, they would get combat powers.
This represented the game jam theme nicely, and we were really happy about the concept.

This time around we actually submitted a working and almost bug-free game, and I really really liked to actually get **feedback**.
With over **30** comments from other participants, on what they liked or what could be improved.

## Takeaways

### Boil it down

We overscoped.
Again.

But a bit less!

I think this was mainly, because we felt we improved greatly since the last game jam, and this one lasted a whole week.

Although it is really hard to submit a 100% complete game, we should have focused on some more important stuff.

The tutorial for the game, was severely lacking, and it is really important as this is what introduces players to the game.
The balancing of the game was as well rather off, with some levels being way easier or harder.

There were some things that we could have cut off to improve on this.

As an example, the trader wasn't really a key element of the game loop, and we had a ton of different enemies.
I could argue that if we dropped these two and searched for quicker alternatives, we would have ended up with a more well-rounded game.


### Playtest, Playtest, Playtest!

We actually planned to do so, but we didn't do it as we decided to cut it off to improve and bug fix some stuff.

When you have been working non-stop on a game, for a week, on a coffee overdose, it is hard to think about your game clearly and objectively.

This is why you need to make other people test your game!
So you can see how other people actually interact with your game, and if they play the way you intended to.

After this, we got some feedback and we realized some mistakes, that we would have liked to fix before the end of the game jam.

So don't wait for the end of the game jam to get feedback on your game!

### Have Templates

This one is a serious speedup.

From the previous game jam, we scrapped together a game jam template.
With a really basic UI, and scripts we knew we would use like a sound manager.

This is really nice as it frees you up from a lot of tedious stuff, that you would almost reuse anyway in every game.

But be aware that templates should be rather compact and modular, so you can quickly build on top.
But most importantly, so that you can keep being innovative and experiment with other stuff, if all your games use a template, and the games look/feel/are very similar, there's a problem.

After this game jam, I will probably be re-tooling some UI, and asset management scripts to re-use for later game jams.

# GMTK 2025  

The Theme was **Loop**.
And it lasted a weekend, with 9,564 games submitted.

This was mostly a warm-up, I learned a lot about **Godot**, game development and design.
The concept of the game was heavily inspired by [Vampire Survivors](https://store.steampowered.com/app/1794680/Vampire_Survivors/) and [Brotato](https://store.steampowered.com/app/1942280/Brotato/) like games.
The **Loop** part, is that the attacks and effects used to combat, are actually controlled by a spinning wheel, and the player had to correctly time it and press **space** to trigger the effects in the wheel.
Obviously there were some effects with negative penalties.

## Takeaways

### Submit Early
We failed to submit the game, so we didn't get a ranking :( .

Either we misunderstood how to submit, or there was a bug in the submission.
So my **tip number 1**, would be to submit, 1 or 2 hours before the deadline, or even days before when you have the first prototype/sketch of the game, so at least you have something to show.

### Project Scope
An important aspect of game jams is having a proper scope of the game, and quickly prototyping.
We failed in both these aspects, which honestly is not a surprise as it was our first one.

When scoping the game, there are many factors to consider, but the main ones are:
- Team Size
- Team Experience
- Available Time 
- Goals (are you here to **learn**, to **experiment** or to **win**?)

The game was way too big scoped for a weekend, and 2 beginners in gamedev.
To top it all off, we didn't prototype enough, we just had the initial idea and ran with it without checking if it was fun or not.

Ideally, one should prototype the base game loop roughly to test if it's actually fun to play.

### Art

This is particularly rough when there's no artist in the team.

What I would advise and prefer to actually have many asset libraries on hand, and then build your game with the assets you have, this way the game art looks coherent.

I understand this can be limiting, but in the case of this game jam, we actually stuck together a lot of different assets, different styles, and it ended up looking rather janky.


