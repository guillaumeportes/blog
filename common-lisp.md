# Using Common Lisp in games

\#common-lisp \#ethereum \#docker \#ecs

## Introduction

This post is about how we use Common Lisp at Tinka, and how we got there. It does not go into implementation details, but rather focuses on high level motivations and consequences of using Lisp.

I have been interested in Lisp languages for a long time. Maybe it comes from having been exposed to Scheme in University (back in 1996 or 1997 at Université de Nice Sophia-Antipolis in France). I kind of forgot about it for a while, as I was learning how to make games. As I was working on client stuff, it was always C++ or C#. Later on, when working on BattleHand, the server was written in Java, which is pretty similar.

A few years ago my interested became more concrete, and I bought [*A Gentle Introduction to Symbolic Computation*](https://www.cs.cmu.edu/~dst/LispBook/book.pdf) and [*Practical Common Lisp*](https://gigamonkeys.com/book/). For a while I kept reading them (especially the former, which I must have read three times), fascinated by what was inside. I also kept reading blogs and articles about Common Lisp, such as the excellent [*Road to Common Lisp*](https://stevelosh.com/blog/2018/08/a-road-to-common-lisp/).

The more I was reading, the more I wanted to play with it. Unfortunately we were making hyper-casual mobile games, and it made no sense to do anything in Lisp.

## Blockchain Pivot

In early 2022 we pivoted away from hyper-casual and into the business of blockchain gaming. While this was ultimately fruitless, it allowed me to finally scratch that massive Common Lisp itch, and I decided that whatever it was we were going to come up with, we will use Common Lisp. There were a couple of reasons behind this decision:

* After having read quite a bit on the language, I felt it would be a good fit. Specifically, the ability to program interactively and to easily write programs that generate code
* I really wanted to have fun and learn. I think we often get bogged down with making the right decision, and I was at a stage where I recognized the importance of having *fun*. I believe programming should be *fun*

We set out to build a prototype for a platform to create, publish and sell choose your own adventure books (such as [*Flight from the Dark*](https://en.wikipedia.org/wiki/Flight_from_the_Dark)). We thought it was a nice blend of content creation and management and gaming, and that it was a nice use of the technology (we were wrong).

On the technical side of things the product had the following pillars:

* A Common Lisp library to interact with the Ethereum blockchain
* A Common Lisp server to allow players to free-to-play the books
* A Common Lisp front end to publish the content
* A Common Lisp DSL to write the books
* A client using Unity

### Docker + AWS ECS

Of course, Common Lisp not being exactly mainstream, we needed to find a platform allowing us to easily run and deploy it. We had used Elastic Beanstalk in the past (with Docker), but I wanted something lighter. I also wanted to make sure I could easily deploy everything from shell scripts / command line.

Using Docker Compose and the ECS integration allowed us to achieve all this. Thanks to Docker, we can easily create an image that runs our Common Lisp software, and compose allows us to deploy (almost) transparently to AWS. Our pipeline looked like the following:

* We push to our repo and Docker Hub builds the containers (at the time of writing, I have not managed to build locally on a MacBook Pro for x64)
* Once it is built, we can update existing environment or create new ones by calling `docker compose up`
* We also create / update the relevant route 53 resources, and enable our container for ssh
* Thanks to Common Lisp, we can easily connect to the instance and change code / recompile / live update what is running

There is of course plenty of details (and could be the subject of a post of its own), but this is the gist of it. What is great is to have a pipeline that allows us to interact with the code in the same way whether we are developing locally, or in a local container, or in the remote container.

### A DSL to generate Ethereum smart contracts

As our goal was to create a platform for anyone to write interactive stories / books running on Ethereum, we needed a pipeline / toolset to do so. One thing that becomes very quickly obvious with Common Lisp is that it is a language that is great to generate code (including in Common Lisp, of course). After having determined how a "Game Book" would look like on the Solidity side of things, I set out to write the DSL in which authors would write their stories. This intermediate language would be consumed by a Common Lisp compiler and generate Solidity code. The platform would then ingest that code and deploy it to the chain. Images tags would be picked up and uploaded to IPFS.

The end result was super interesting, with non technical people able to write the following:

```
(section 1
  (text "You must make haste for you sense it is not safe to linger by the smoking remains of the ruined monastery. The black-winged beasts could return at any moment. You must set out for the Sommlending capital of Holmgard and tell the King the terrible news of the massacre: that the whole élite of Kai warriors, save yourself, have been slaughtered. Without the Kai Lords to lead her armies, Sommerlund will be at the mercy of their ancient enemy, the Darklords.")
  (text "Fighting back tears, you bid farewell to your dead kinsmen. Silently, you promise that their deaths will be avenged. You turn away from the ruins and carefully descend the steep track.")
  (text "At the foot of the hill, the path splits into two directions, both leading into a large wood.")
  (transition 141 :text "If you wish to use your Kai Discipline of Sixth Sense, turn to 141." :ability "Sixth Sense")
  (transition 85 :text "If you wish to take the right path into the wood, turn to 85.")
  (transition 275 :text "If you wish to follow the left track, turn to 275."))
```

A book would be made of multiple sections and transitions, fights, items to pick up, decisions, etc. The above code would result into Solidity code that would allow users to play the book on the blockchain (the Solidity code was, obviously, far less pretty than the DSL one!).

Interestingly, we were able to get ChatGPT to create content by simply explaining the syntax of the above language. The content was, of course, not of the best quality!

## Mana Storm

Unfortunately, as interesting as the game book experiment was, it became clear at the end of 2022 that we would not be able to raise funds for it.

After some down time and research, we decided to focus our efforts on a PvP card game similar to Hearthstone / Magic: The Gathering. The game is split into two parts:

* A client written in Unity / C#
* A server written in Common Lisp

Of course we were able to reuse most of the infrastructure related stuff we had written for the game book project, i.e. the scripts and code related to deploying a CL application to AWS ECS.

I do not want to get into too many details (again, probably best for further posts), but apart from the fact that it is a hell of a lot of fun to write a game in Common Lisp, there are a couple of interesting patterns that it allows.

### A different kind of component system

A very common pattern in games is the one of components: typically, in order to add behaviour to an object in the game, one adds a component to it. That component defines certain functions that implement that behaviour (flying, shooting, etc.). In Unity, this is exposed inside the editor, where one can create complex objects by combining components on them.

One of the things it allows is to have composability without using class inheritance, which is limited given the problems around multiple inheritance (absent in C#, messy in C++).

In Mana Storm, some abilities can be thought of as components: for instance, if a card has *Bait*, others are forced to attack it. *Deadly* means that any damage from it is lethal.

Thanks to Common Lisp, we are about to do the following, which blows my mind:

* When adding a component to a card, we create, at runtime, a new class. This class inherits from the card's current class, as well as the component we are adding
* We then change the class of the card instance to the newly created class
* Similarly, when removing a component, we change the class to one without that components (if it does not exist, we create it like in the adding case)

What it allows us to do is have a flexible component system that still allows to simply overload the same methods, and therefore rely on the language to call the right function depending on the type of the instance.

### A language to describe cards

One of the important challenge in making a card game is to be able to make original cards. One of our previous games, BattleHand (which is set in the same world as Mana Storm), was very data driven in that regard, and it quickly became difficult to make cards that were different from previous ones.

In Mana Storm I knew we needed cards to be code. But I of course did not want to have to write the same stuff over and over again. I was able to quickly and easily write a language abstracting much of the details of how a card works internally, and focusing only on what it does. This is similar to the language generating Solidity code described above, the difference being that this is Common Lisp generating Common Lisp.

In the end, we have a pipeline allowing us to create cards very quickly (we often have a card up and running in 10 minutes, including tests), without sacrificing depth of content.

Here is an example:

```
(defcard gargoyle (<minion>) (:data-id "40a234a9023b6a696cb2a3140cdaa994"
                              :rarity :epic
                              :health 2
                              :power 1
                              :cost (:spirit 1 :colorless 2)
                              :tags :bat
                              :display-name "Gargoyle"
                              :description "Gain +1/+1 when an enemy minion dies.")
  (on-death (card)
    (when (and
           (battlefieldp #?gargoyle)
           (enemyp card #?gargoyle)
           (plusp (health #?gargoyle)))
          (modification/health+ 1 :origin gargoyle)
          (modification/power+ 1 :origin gargoyle))))
```

As you can see, this is very expressive yet succint. This of course expands into a lot more, including the definition of a class for that card (in order to overload methods).

## Conclusion

I think investing time and effort in Common Lisp has given us new tools in terms of how we can solve certain problems. As it is such a different language than the usual suspects (at least in games!), it has been both great fun and enlightening.

I also believe that it gives us a competitive advantage when it comes to a game like Mana Storm.

A while ago I read a blog post (or maybe was it on Reddit) where it was suggested that learning Common Lisp was not for the above 40 programmer. I think it could not be further from the truth (I am of course in that category!). If anything, I think Common Lisp is the language of the mature programmer: the programmer who cares about getting things done in a pragmatic fashion. It feels to me Common Lisp does not try to force you to do anything, it gives you the tools to do it how you want. I love it!
