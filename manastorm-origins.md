# The birth of Mana Storm

[Mana Storm](https://manastorm.tinka.games/play) is a collectible card game for browser in early access. What follows is the process we went through to come up with it.

## Introduction

In November 2022 Jérémie and I went to Slush to pitch our grand idea for a *Choose your own Adventure* blockchain publishing platform to investors. For months we had been drinking our own kool-aid: we had convinced ourselves that making games on the (Ethereum) blockchain (I am talking about on-chain games here, where the game actually runs on the chain) was the next big thing, and that our idea was not only cool but highly investable.

That trip was tough, as it made us realise we were very wrong (sorry, that is very, very wrong): it was clear that while the various VCs we met enjoyed our conversations, no one really believed in what we were pitching. A lot of it had to do with the fact that, well, not enough people care about interactive stories (at least in that context).

We came back feeling convinced we had to pivot the company yet again.

## Independence

We felt that a few times over the lifetime of the studio (we founded Another Place in early 2012 and rebranded to Tinka in 2022) we made decisions a bit too quickly. As a result we decided to take a bit of time and properly think about our options. At that time (December '22) we had about a year of runway, and it was important to make the right decision.

Tinka has had the following "phases":

* 2012 was generally dedicated to making mistakes and culling the founding team (if only subconciously)
* In 2013, as we were running out of cash (this is a recurring theme 🙂) we made Dragon Finga, a free-to-play mobile game
* In 2014 we secured about $1M in investment and built a team to make a large free-to-play RPG, [BattleHand](https://www.youtube.com/watch?v=jz2r4Ovu1UI) (unfortunately now defunct)
* 2018 saw the release of BattleHand Heroes, with disastrous results for the studio (we knew the game was going to fail, but publisher pressure meant we still finished and released it, and had to lay off half of the staff in the process)
* We made [hyper casual games](https://medium.com/@guillaumeportes_18178/the-tough-business-of-hyper-casual-games-fc59b775beaf) from '18 to '21
* And finally tried to pivot to blockchain stuff during 2021

When looking back at those phases, we realised that we often struggled to fulfill one of the core reasons we setup the business in the first place, which was to stay independent. Sure, we never worked for anyone, but we almost always worked with publishers with sometimes damaging results (the economics of giving away 50% of your revenue on top of the 30% that the platform holders take makes it very difficult to sustain a business unless the game is very profitable).

On top of that, we had the terrible experience of having a successful game cancelled by Apple.

We came to the conclusion that one thing of utmost importance to us was independence. From that conclusion we naturally decided to make a browser game. That way, not only would we be able to self publish, we would be able to self distribute.

## The genre

With the platform (browser) decided upon, we set out to figure out what kind of game we would make. We quickly settled on a few fundamental pillars:

* It would be a free-to-play game: we have been immersed in F2P for more than a decade now, and we love the business model. I know it gets a terrible reputation with some people, but to my mind it is the one that maps the best to long tail games (which we want Mana Storm to be)
* It would be synchronous multiplayer. This is not something we have that much experience with (in terms of released products), but we feel this is the key to build a long lasting game: it allows us to focus entirely on the core mechanic without worrying about players running out of content
* We wanted to build a core game. After years making hyper casual games and trying to make games for a wider audience, we wanted to be able to focus on the core content, and make a game for a very specific audience

With those in mind, there were a couple of obvious models to us:

* Hearthstone: we had played Hearthstone quite a bit when it first came out, and of course all loved it. It was around the time we had started making BattleHand, and although very different, it was still a card game
* Clash Royale: that, we played solidly for a very long time, 5+ years. We actually had prototyped a similar game around 2017, but never managed to launch it

In the end, the decision was technical: I felt it would be much easier to prototype and make a synchronous turn based game. We wanted to release very early and build the game alongside the community (thanks to our hyper casual experience), and I knew we would arrive there quicker that way.

In addition, we had a lot of content and lore from BattleHand that we knew we could easily reuse: characters, cards, monsters, etc.

## The game

The next stage was to figure out what we could do to make Mana Storm different enough from Hearthstone. We spent a few weeks studying the market and deconstructing the myriad of card games out there. While we wanted to make something different enough, we were concious of trying to innovate too much. I know many people look down on this, but I do not believe in innovation for the sake of it.

I played a bit of Magic: The Gathering back in the 90s, and I thought it would be interesting to combine the core mechanic of Hearthstone (that is, the combat mechanic) with the mana system of MTG (that is, different mana colours and total freedom).

A couple of reasons fuelled this idea:

* In our opinion, Hearthstone plays much better than MTG: Arena (to be crystal clear, I am talking about the digital version). It is far smoother. Playing MTG: Arena feels very disjointed, with many interruptions. In contrast, when it is your turn in Hearthstone, only you can interact with the game.
* While we understand the idea behind character specific cards in Hearthstone, we feel it comes with restrictions that we want to avoid in the long run (this is from first hand experience: in BattleHand, we had class and "color" specific cards, and it became a nightmare in the end). In contrast, the mana system from MTG gives an incredible amount of depth. Of course the cost is a lot of complexity on the player's side, but I feel this is good complexity, i.e. depth

As we could not find a game combining those elements, we set out to make that game. A synchronous, turn based CCG with Hearthstone combat, MTG mana system, set in the world of BattleHand.

## The tech

The technology side of things was pretty easy, in terms of settling on what we would use:

* We have been working with Unity for 10+ years now, and it was a no brainer to carry on doing so for the client
* Server side, I was very keen on carrying on with my [Common Lisp](https://medium.com/@guillaumeportes_18178/common-lisp-in-games-bd7c87f1446a) adventures. The game obviously needs to be fully server authoritative, which meant writing the core of it in Common Lisp. The client would "only" deal with presentation and interface
* Regarding infrastructure, we had been using AWS for a while (10 odd years again, actually), and we had a pipeline for deploying Docker contenerized Common Lisp applications in ECS, so there was no need to look further

The key thing we wanted to implement (beyond obviously making a solid, beautiful and responsive game) was to have total freedom when it came to designing and making cards: in BattleHand, I had written a system I liked to liken to Lego blocks: each card was made of components that the designers could combine however they wanted: "damage", "heal", "poison", etc. Combining "damage" and "heal" was "leech".

I initially thought it would give them great flexibility, but in the end it was quite restrictive.

Looking at Hearthstone and MTG, it was clear that cards had very, very custom content. It will be the subject of a separate post, but in Mana Storm defining a card is done in code, using a domain specific language hooking into the various events dispatched by the game.

For instance, here is the code for "Despotic Bat":

```
(defcard despotic-bat (<minion> <targeting/play/minions>) (:data-id "456b42cc376081dbf00c624110f01250"
                                                           :rarity :common
                                                           :power 2
                                                           :health 1
                                                           :cost (:spirit 1 :colorless 1) :tags :bat
                                                           :display-name "Despotic Bat"
                                                           :description "<b>Warcry</b>: <b>Mute</b> a minion.")
  (on-played (despotic-bat :target target)
    (when target
    (mute despotic-bat target))))
```

Very succing, expressive, and incredibly flexible.

## The go-to-market strategy

Finally, we had to figure out how to get to market. This is obviously something we are still figuring out, but the broad lines are that we want to grow the game organically and as independently as possible (things can change though, as I write this we are investigating the Epic Games Store). We also want to make the game alongside the community (you can find our Discord [here](https://discord.gg/EZaNdYDp)), and we released the first version last June.

This is why we are now working hard on trying to create interesting content (including hopefully those posts).

Anyway, we are at the start of that journey, and hopefully we will manage to grow Mana Storm into the big game we believe it deserves to be!
