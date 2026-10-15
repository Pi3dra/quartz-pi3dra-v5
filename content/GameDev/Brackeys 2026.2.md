# Brackey's Game Jam 2026.2 

<div style="text-align: center;">
  <img
    src="/images/2048.gif"
  >
</div>

The theme was **Trust No One**
It lasted 1 week, with 2,245 games submitted, and I finished with an overall **rank of 175**, climbing up almost a 100 places compared to last year !

This year I got way more reviews than last year: With over **50** comments from other participants, and **77** reviews!

You can **check it out** [HERE!](https://pi3dra.itch.io/2048-uncertain-ones)

This time I was alone, as the person I often do game jams with wasn't available, so I decided to go with something really simple.

This time around I actually submitted a working and almost bug-free game, and I really really liked to actually get **feedback** on the mechanics themselves.

# The Game Idea

The idea itself is rather simple.

It's the basic 2048 game, but all tiles containing a 1 (16, 128, 512, 1024) have a random effect when they are played. 

These effects could be really simple like winning some extra points, or really hard consequences like shrinking your grid.

To get more information you could use the **magnifying glass**, to get more or less useful clues about what the tile may or may not do.

If you are certain the tile is bad (or you don't want to risk it) you can shoot at it with the **gun** to disable it's effects. For skilled players if you merge a good tile with into a bad tile, this will also negate the effects of the bad tile.


# What I've learnt

## Test Your Interfaces/Interactions
Test your UIs, 2048 has very very simple controls, **4 keys, up down right left**. 
Suddenly now, players expecting a simple 2048, found themselves breaking out of the classic game loop to:
- **BUY** items through the shop.
- **SELECT** the **ITEM** they want to use 
- **SELECT** the **TILE** they want to have an effect on.
And this multiple times as often they could ask multiple hints from a single Tile.


So a few things to consider about interactions.
- How key is this action? (Is the player supposed to do it often)
- In which contexts this interaction occurs? Is the player in a hurry? can he sit and think about it?
- If this interaction is skipped what are the consequences?
- How many actual interactions this takes to trigger? count actual keystrokes, clicks etc etc.


In my game, the **Magnifying Glass** needed to be used many times 3-5 times to be absolutely certain a tile isn't bas.

**How key is it ?**
It's actually central, as it is one of the main ways the player has to know which tiles have good or bad effects, it is expected to be done often.

**Context?**
The player encountered one or more tiles, and he doesn't know their effects. It is a 2048 games, players can sit and think if they want, but it is a simple game that is expected to be playable in a quick manner.

**Consequences of skipping it?**
The player exposes himself to the risk of triggering bad tiles. In short term the gambling aspect can be fun as they may have good tiles, but then makes the game boil down to sheer luck.

**Actual Interactions?**
So if you have 0 **magnifying glasses** available, the player has to:
- Drag Mouse to **Buy** button
- Click 5 times **Buy**
- Drag Mouse to **Magnifying Glass** 
- Click **Magnifying Glass**
- Drag Mouse to the **Tile** we want to inspect
- Click 5 times to trigger effect.

This makes a total of : 14 interactions, to have access to a main mechanic.

### Conclusion on Interactions

Once the previous assesment is done, it is up to you, to accept it as it is, or fix it if it's broken.

In my case this was a core mechanic, meant to be used often so it was crucial for it to be as smooth and easily accesible as possible.

The solution I would have implemented now:
- Get rid of the shop, charge players when they click on a tile, basically auto-buying
- Use left and right click for the Magnifying Glass and Gun respectively

This would have made it so a simple drag and click easily triggered a power.

## Quick Thoughts On Shops

>[!tip] TLDR
> For small prototype game jams, a shop is often not the best choice. Unless it is the main mechanic of your game. 
> If you feel obliged to use this mechanic nevertheless, keep it as simple and transparent as possible.

Shops are a very complex subject, and I would really like to dedicate a whole post talking just about it. But here's my quick opinion on shops.

In their simplest form, shops allow players to exchange some sort of currency for benefits, being items, powers or skills. 

### Functions Of Shops & Alternatives
Shops can serve many functions:
- Reward players for their hard earned coin
- Establish a certain progression pace
- Show players powerful new items and hint them towards a goal
- Help players out of sticky situations (Mainly in roguelikes)
- World Building, and immersion

Considering all these functions, it is easy to see why implementing such a system for a game jam entry is complex.
To top it all off, these functions can be done in better ways, through simpler systems.

**Rewards** can be given per level, per score, per progression in the story and stuff like that, setting a wanted progression pacing.

**Helping** players, can be done through pity systems or more impactful mechanics like finding hidden items.

**Showing** players powerful items can be easily done through world building and story telling, make your boss drop a powerful weapon instead of buying it somewhere, and the player now feels like he's as powerful as the boss (and you get some free asset reuse).

I think the previous examples, may be a bit vague, as they depend on the type of game you are doing. But in my opinion it gives more character to the game, as well as ways to add more immersion to your games. 

### Common pain points
What I am trying to get at, is that a table of items with a price tag, can be less way impact-full and sap the fun out of your game, if the shop is implemented without that much thought.

A shop introduces an **economy system**, which can be simple, or very complicated, whichever the case, they require careful balancing. Make it too expensive to get something, and players may ignore your shop altogether, make it too cheap and players will exploit this to break your game. (It is interesting to note that "breaking" game mechanics can also be fun by their very nature, I'm thinking about **Balatro** here).

**Spreadsheet UIs**, no more, no less, boring interfaces, meant to get the main function of the shop done, without any story telling or player immersion. This introduces tedious menuing and an overall worse experience.

**Context and pacing problems**, give the player access to a shop with all the items in the game, he is overwhelmed, and the element of surprise of new items is lost.
Randomly generate shop contents, and the player may not want or need any of the items. or get an overpowered item to early in the game

**Artificial Gate Keeping**, put a price tag too high, and players may need to unnecessarily grind to progress in the game, pads playtime, but doesn't add any real value to the game. This may or may not reward the most engaging types of play, and introduce boring grinding.

### Conclusion on shops

So to sum it all up, It may seem that shops are a bad game mechanic after all the previous points, but if it is executed correctly, it can be a key and fun element of your game.
I just think that they aren't a good asset for game jams.

They are complicated to implement in a meaningful way.
There are many alternative mechanics that may serve the same functions in a more immersive and impacting manner.
And finally game jams are about testing original and creative ideas, and shops aren't any of both.

## Thoughts On Randomness

Now onto randomness, my two game jam games have randomness as a key factor of the gameplay.

I have been thinking about this a fair amount, as I don't like the idea that someone who is really good at the game might still loose because of bad luck.

There are two types of randomness.

- **Input Randomness** : The game gives the player a random situation to handle.

- **Output Randomness**: The player executes an action, it has % chance of playing to his advantage.

> [!info] Notice that output randomness feeds into input randomness as the output can determine the situation which the player might need to handle next.

### Randomness in my games

In Risky Cookies it was mainly **output randomness**, as the result of the cookie(coin) flip, was determined how well you did in combat.

In this game jam, there was a mix of **both**, there was **input randomness**, as a consequence of suspicious random tiles spawning. On the other hand, the player encountered **output randomness** when he asked for a hint, which could vary in quality. from very ambiguous hints, to precise hints which unveiled exactly what the tile did.

### Functions Of Randomness

Again, let's do a non-exhaustive recap of the functions we expect from RNG.
- Procedural Generation and Variety
- Balance Multiplayer Games
- Reward Systems
- Non-Player Controlled combat systems, ex: damage and dodge chances in turn based games.
- Avoid players optimizing plans to a point where they aren't challenged (surprise factor)

### Consequences of Both Types

#### Input Randomness
It is very interesting to note that this type of randomness can have little impact on actual gameplay.
World generation in Minecraft dictates how the world will be, but it doesn't significantly affect how the game is played. The world is so big that the player is certain to find a place he likes, and is certain to find good biomes and resources.

And in the other hand can have a huge impact, getting good loot at the start of a game, can throw the game out of balance, and break pacing. This would encourage players to restart the game until hitting a good starting condition.
Note this can be fixed by adding systems that encourage the player to at least try. ex: Item Stashes in White Knuckle to encourage users to try the start of the game, and leave good loot behind for next runs.

#### Output Randomness
This one can easily make or break a game. 

It is often tied to risk, it helps to build suspense, and holds players in a state of tension when high-stakes rolls are in play. Which as a consequence promotes risk management, as the player may be in a dicey situation if his plan goes sour without a backup plan.

If it is badly implemented, or just a yes or no output, it can severely hurt strategy. by making the player plan out a perfect move and still loose.

It creates unexpected and satisfying narratives, ex: that one time you had 1HP and pulled out a critical attack on a boss, dealing a fatal blow and winning the game. 

#### Global Consequences
It is also interesting to consider the frequency on which random events happen. 

If your turn-based game has 0 randomness, the player doesn't need to plan ahead in case something goes wrong. Make randomness too frequent and the player may feel like whatever his plans are, it's only a matter of luck.
If frequency is correctly balanced, players will have enough time to build a plan, as well as enough time to adapt to unexpected situations.


## Making Good RNG Systems
Here are a few bullet points.
- Implement Pseudo-Randomness, make it adapt to extremely lucky and unlucky streaks (It can make the game more accesible, but give the game a more aggressive and rough edge if not implemented, which can appeal to a certain public and narrative)

- Provide Mitigation Systems, add alternative mechanics so that players can exceptionally bypass luck. (Shops can do this, as well as special buffs)

- If output randomness is used, it is best if it either benefits the player or it does nothing. ex: Blocking/Dodging in Darkest Dungeon. It is extremely rewarding when you think something is going to go wrong, but by a bit of chance it is avoided.

- Make output randomness, not a binary output, but a gradient, to make it less punishing. And always choice the chances.

- Provide systems that encourages players to not restart on bad input luck.

- Adapt the frequency of random events so that players have time to plan out and adapt.

- Make the RNG a concrete mechanic instead of a plain percentage, Humans understand better dice rolls, card hands and coin flips.

- Note if your randomness is dependent or independent, pulling cards change the probabilities of your next hand, but rolling a dice has no impact on the next dice roll.


### Conclusions on Randomness

We can conclude that randomness can make or break a game. And there are few alternatives that could serve the same function in games.

I think that for small game jams it is rather hard to make a good RNG system, but it may be worth the shot if you think it will benefit your game. 
Now knowing all the functions and consequences of randomness one should make an informed decision and decide **if** and **how** to build an RNG system.

The main Takeaway is to be conscious of the type of randomness you are using, and adapt it to avoid the game feeling like a coin flip. Instead randomness should provide new scenarios to surprise the player, and make him adapt to new information, as well as managing the risk of his actions.

## Final Conclusion: Doing An Old Game With A Twist

To develop games based on top of classic games, there are some things to take into consideration.

There's a high chance players know this game, change it too much and you break user experience.
They expected to play 2048, but suddenly they found themselves shooting at tiles.

The idea is to add your new mechanics in the most seamless way possible. by making new mechanics not disruptive to the original gameplay loop, and also respecting the visual and audio identity of the game.

In my case, the complex interactions significantly broke down the gameplay flow.

The randomness of it all tied as well to the theme, but the output randomness of the lookup system made it tedious to strategize. Maybe making the tiles reveal their effects and give the player a certain amount of turns to handle them might have played out in a better manner.

Also I personally found this approach to be a bit boring, as it didn't feel very creative and original, but it is a very good way to:
- Analyse classic game mechanics
- Quickly get started
- Get your hands into programming anything

# Takeaways

- Test thoroughly your UIs and the concrete interactions behind mechanics.
- Think twice before adding a shop to your game, does it make sense interaction and mechanic wise?
- Map out which type of randomness you use in your game, and how it changes gameplay.
- Making an old game with a twist is a good way to get you started, but extra care should be done to add mechanics in a seamless manner.

### Sources:
[The Two Types of Random in Game Design - GMTK](https://www.youtube.com/watch?v=dwI5b-wRLic&t=88s)
[How Video Game Economies are Designed - GMTK](https://www.youtube.com/watch?v=Zrf1cou_yVo)
[Output Randomness: Game Designers should know how to use it. - Ezra Szanton](https://ezraszanton.substack.com/p/output-randomness-game-designers)
[What makes a great in-game shop? - Design Doc](https://www.youtube.com/watch?v=B_pfFVXBvic)
