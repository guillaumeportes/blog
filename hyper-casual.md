# The tough business of hyper casual games

## Introduction

In May 2018, as we prepared to release BattleHand Heroes, Another Place (which is what Tinka was called back then) was in a tough spot: we had very little runway, and there was nothing obvious to fall back on as we knew BattleHand Heroes would be a commercial failure.

A long term advisor suggested that we pivot to making hyper casual games. We immediately loved the idea for a few reasons:

* We were tired of working on large projects (18 months minimum) and the prospect of rapid prototyping was appealing
* It was exciting to try something new and different
* It felt like the only option that allowed us to have multiple shots at goal in a very limited time (6 months)
* It is an extremely metrics driven business. This does not necessarily go against creativity, rather it makes defining success and failure very clear: the metrics a game must hit for success are well defined (although they do change as the market evolves)

We effectively pivoted in July 2018, after releasing Heroes and doing a tiny bit of live operations on it.

## Learning

The first thing we needed to do besides the obvious market research was to learn (and unlearn) a lot of things: while we never considered ourselves old school, we were definitely a team with strong AAA roots (we all had worked at Lionhead Studios on the Fable series). This comes with a lot of advantages (high quality, know-how) but also a lot of drawbacks (fear of release something that is not perfect, for instance).

From the technical side of things, things seemed easy: the games were simple, and the challenge was mainly about establishing an efficient pipeline to automate the process of creating new projects:

* New repository in GitHub with a skeleton code base
* Creation of the Unity Cloud Build project
* Creation of the game on the App Stores (iOS and Google Play)

This was done on the command line, through a combination of Python, NPM (I had a phase of writing scripts with package.json) and Bash scripts. Thanks to this process we could kick off new games within 5-10 minutes without having to do any manual work.

Similarly, on the Art and Animation front, things looked very simple, at least at surface level.

As often, the real challenge was with Game Design, and it was twofold:

* We wanted to release a minimum of one game per week (for reference, we were a small team, no more than 6 people involved in the process)
* No one in the team was an actual "Game Designer", and we had to prove (including to ourselves) that we could sustain this output

One thing that a lot of developers initially think when looking at HC games is that they are, well, shit. At Tinka we have always been humble (numerous failures have helped keep it that way!), and we thought that commercial success meant that *something* was good about those products. The challenge was to figure out why.

We even realised down the line that there are most times very good reasons why bad looking art styles actually work better for these games.

We quickly found that coming up with little games was actually not that hard, and that once one understands it is about releasing, testing and failing early, it it similarly easy to let go of them once it is established they do not work.

A lot of our first few games were variations of successful ones in the space (the mighty [Helix Tap](https://apps.apple.com/be/app/helix-tap/id1406877090) for instance), but a fair few were more original (like [Roll On](https://apps.apple.com/be/app/helix-tap/id1406877090)). What was great was that everyone in the team became a *Game Maker*, which was very empowering. Everyone became able to design and create games by themselves (without it becoming the norm, actually).

We also learned a lot in terms of digital marketing, and it became clear a few months into the process that to make any money in this space, we needed to work with a publisher (which did not thrill us). Luckily, we were making [Mow.io](https://apps.apple.com/be/app/mow-io/id1439915524) at the time, and the potential of the game meant that we managed to get a contract to make games in partnership with [Lion Studios](https://lionstudios.cc/). This partnership started in January 2019.

## The marketing approach

Working with Lion Studios was great fun: we were already hitting the expected output (both in terms of quality / relevance and pace), and were now exposed to a lot more data: the main difference in terms of market research is that those guys were able to look at games that had not been fully released yet, or games that failed but were interesting. As those games were way down the charts, this is not something we were able to do by ourselves.

That being said, there was not much method to their approach beyond trying as many things as possible: there was no formal process to coming up with an idea - everything was on the table. We made very different games with Lion (which was part of the fun of course):

* [Mr Pole Vault](https://apps.apple.com/be/app/mr-pole-vault/id1451610326), a weird rag doll based physics game. This had great initial marketing metrics, but we could not follow up in terms of retention
* [Fashion Inc.](https://apps.apple.com/be/app/fashion-inc/id1449723782), strongly inspired by [Idle Coffee Corp](https://apps.apple.com/us/app/idle-coffee-corp/id1441021951). Clearly not a hyper casual game, which was reflected by the long development time (two to three months)
* [Save Them All!](https://apps.apple.com/be/app/save-them-all/id1482615457), a great physics puzzle with a drawing mechanic

Unfortunately, while we felt we learned loads, there came a point where we had to cut ties with Lion, and look for a new partner. We started working with [Coda](https://codaplatform.com/) in January 2020. Again, we had a great relationship with those guys (their CEO is now an investor in Tinka), and their approach was similar to Lion's: both publishers had very strong marketing "origins" (Applovin / Adcolony). The name of the game was really to identify trends and successful competitors, and try to replicate what they were doing.

But alas, after a few months, we still could not get a game to the right level of metrics.

## The methodical approach

In June 2020 we started working with [Voodoo](https://www.voodoo.io), who at the time were the leaders in that market (they may very well still be). The way we started working together was very strange: for a few months between the end of 2019 and mid-2020, we were in discussion for them to acquire Tinka. When it did not happen, we had the opportunity to work together. We decided to swallow our pride and disappointment and started the relationship in June 2020.

It became very quickly apparent that Voodoo had a different approach. Theirs, while still very influenced by marketing considerations, put *gameplay* at the core of ideation and prototyping. There was a method to coming up with ideas, and a method to implement them in the best way. It was fascinating to be immersed in such a depth of analysis on seemingly super simple games.

What I loved with Voodoo is that they constantly challenged our approach with new methods, pushing us to be more formal in our process when coming up with ideas. There was no more picking a game because it sounded cool. It had to satisfy much stricter requirements. As a result our games became more original: sometimes in the theme, sometimes in the core mechanic, etc.

Here are some of my favourite examples of this period:

* [Body Morph Race](https://apps.apple.com/be/app/body-morph-race/id1599480095), a Mario Kart inspired race where the player can sculpt his racer. I was convinced this would be viral as the potential for laughter was huge
* [Bungee Balls](https://apps.apple.com/be/app/bungee-balls/id1590297868), a runner with a cool and exhilarating elastic band based core mechanic
* [Peel Hill](https://apps.apple.com/be/app/peel-hill/id1560677761), a merge between Dune / Bike Hill and Spiral Roll
* [Spiral Roll](https://apps.apple.com/be/app/shortcut-roll/id1564104841), a race mixing the Spiral Roll core mechanic with Shortcut Run

As always we were executing at a very high quality level, and those games were taking between one and two weeks to develop (most games being made by one engineer and one artist).

## Fat 2 Fit

In May 2021 we had a breakthrough. One week-end, while mindlessly looking at Facebook, I noticed I was hit by a lot of fitness apps. They all had a similar layout and message: a fat body becoming slim. I realised there were not many themes more mainstream than fitness. One of the challenges of hyper casual is making a game that the widest audience possible might want to play. This is one of the key to low CPI, of course. It sometimes feels crazy, to the point that a football game is not mainstream enough (as *some* people are turned off by football).

But everyone wants to get fit. We built the game very quickly, as it was a fun but super simple concept. [Fat 2 Fit](https://play.google.com/store/apps/details?id=com.anotherplaceproductions.fat2fit&hl=en&gl=US) was a runner where one needed to avoid burgers (which made you fat) in order to be able to go through the obstacles. The marketing results were immediately amazing. The effective CPI was so low it was "irrelevant". By that I mean that the game was so viral, the test CPI was as low as 0.1 cent.

The decision was quickly made to release the game as quickly as possible in order to ride that virality wave. We also had the challenge to go quicker than studios cloning the game (this happened pretty much straight after the initial release), although we were not overly concerned by that: we knew exactly what we wanted to make, and were very confident in our ability to go faster, which we did.

We iterated on the game for two to three weeks, and it was deemed ready for global launch. We iterated further on it for a few weeks.

Launching the game (as in *properly* launching it) was exhilarating. We worked really well with Voodoo, and we never felt they did not earn their share of the profits. At peak they invested 6 figure budgets in daily marketing campaigns, which is obviously something we would have never been able to do ourselves.

Unfortunately the game's success was cut short and ended up even more ephemeral than what is usual in the hyper casual space. We never intended the game to offend anyone, and felt the correlation between eating burgers and putting on weight was not exactly controversial. Unfortunately, the game has a few negative reviews (some of them pretty silly "My sister stopped eating burgers because of your game"). Some people were really angry, and Apple did something Voodoo had never experienced before: they removed the game from sale without any notice, just as it was sitting at number 2 of the free charts. Of course, we managed to put it back up (with significant changes) quite quickly, but the damage had been done. It never really recovered, and we probably lost 50% of revenue. It was very disheartening to have our wings clipped in such a way when we had finally found success.

## The end

We felt that the success Fat 2 Fit brought was what we needed to level up our studio and we were confident we were going to manage to repeat that success within months.

Unfortunately this was not to be the case, and nothing we did after even came close to hitting the right metrics. What was particularly disheartening was that it was the case both for marketing metrics (i.e. CPI) or product metrics (retention, play time, etc.). We tried for a while to hire different profiles in order to plug what we perceived as gaps (a Publishing Manager, as we were incredibly impressed with the ones we worked with at Voodoo, and more engineers, to be able to experiment more with software as opposed to on paper), but never managed to find or land the right candidates.

At the end of 2021, we came to the conclusion that it was time to try something new: we loved our time in hyper casual, but in the end we never felt in control enough: in a few instances it did not feel we could explain why certain prototypes had failed, which, when you are focusing on repeating success, is a problem!

## Conclusion

I am very proud of what we have achieved: we have made strong connections with important players in the business and we have produced one truly successful game. More importantly to me, we have let go of a lot of hang-ups associated with AAA development, and learned and trained to be able to produce (fairly) original games every week, with a strong focus on the core mechanic.

These are tools that we are using constantly. The most obvious benefit being that we have become very good at identifying what is important to focus on, which is not so easy when it comes to making games!
