# Using Generative AI for game AI

At [Tinka](https://tinka.games) we are currently making [Mana Storm](https://manastorm.tinka.games/play), a card battler merging Hearthstone and Magic: The Gathering mechanics.

We released an early version of the game fairly quickly, and initially only supported PvP (i.e. there was no AI at all). The hope was that we would be able to attract enough players from BattleHand (the previous game in the franchise) and grow the game organically from there, with players organising their matches on [Discord](https://discord.gg/tYgCq7BT7A).

In hindsight it was the wrong decision: I think the flaw was that while enough players came to the Discord server, not that many ended up trying out the game. I think most expected a mobile game and were not too interested in a browser one, or maybe simply expected a different game...

Regardless, we then of course decided to implement an AI to drive the game bots.

## Requirements

The requirements were fairly easy to narrow down: in the short term, we needed to provide players with someone to play with at all times. In the medium term, we needed a platform to build our single player "mode" on (by this I mean whatever our onboarding is going to end up being).

We basically needed to have something good enough to 

## Approaches

I thought there were broadly two different approaches:

* Implement a traditional AI

