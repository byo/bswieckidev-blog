---
title: "K-DAG University"
date: "2019-07-14"
tags: ["k-dag", "ideas"]
---

Last time I briefly described the knowledge in the form of a Graph structure.
Each term having set of prerequisites is a node in this graph, each
dependency is a directed edge in such graph.

Now let's assume this graph is a directed acyclic one and there are no
terms which form a cycle of dependencies. In reality I don't believe
this to be true but it does simplify reasoning and can definitely be achieved
locally for larger terms.

Let's imagine that we could create a new tool to help people learn new things
by using such graph.

The shape of University
-----------------------

We've got used to our school systems - we learn in classes with other students,
each year is split into two semesters, there's usually a teacher and students.
There are certain rules applied which meaningful to organize the learning
process and make it efficient.

But let's think about a bit different shape.

- Instead of a teacher we have a web-based Knowledge DAG application
- Instead of attending classes about some topics, people try to gain
  knowledge related to specific precise terms (i.e. Geometry Basics)
- Every student has a set of knowledge points he has mastered,
  some kind of digital signing could be used as a form of proof of
  accomplishment
- Knowledge points (points in the DAG) may be of different kinds:
  - Some knowledge points are milestone ones and group together smaller
    terms (like mathematics groups geometry, trigonometry and any others)
  - Some knowledge points are sequentially chained - i.e. steps to achieve
    one greater point but split into smaller parts to ease the learning
    process
  - Knowledge point may describe some short topic - i.e. 5-minute
    lecture with easy to understand definition
  - Knowledge point may also be some sort of example to clarify previously shown
    term
  - Knowledge point may be an exercise that helps memorizing a term
  - Knowledge point may also be a test that has to be passed in order
    to mark the point as mastered (this I believe would require
    some digital signature to prove the test has been passed)
- The form / kind of knowledge points may be freely adjusted to the
  knowledge that needs to be presented. Some terms may require a lot
  of examples and exercises to be mastered, some could require only a
  brief explanation, some may present a lot of terms which have to
  be mastered one by one
- Some major terms could have different set of "paths" (or
  rather subgraphs) that let master a term. Those paths could
  be prepared by different people, could be adjusted to different
  backgrounds, even cultural differences. The rule is that those
  should be equal regarding the final goal
- The whole graph should be clustered into larger groups
  to help better organizing the knowledge. A group should touch
  one specific bigger topic of the knowledge (just like one class
  in a school). Each such group should have at least few
  major terms in it, preferably one.
- If one group does rely on a knowledge from another group it's best to depend
  on major terms from other groups only. That way we just shape the overall
  requirements but we don't exactly specify how one needs to get to them.

How this could work in practice
-------------------------------

Let's say I want to learn the best way to write some web app in *node.js*. I log into my
"University" page and search for *node.js* term. Once found, I check if it's
a pretty recent one - I don't want to learn something that was relevant 3 years ago.

Once selected, the system shows me few alternative paths which I can choose.
For each path I could see the popularity, my current progress in each alternative
(well, I've learnt some javascript already so it won't be needed here), info about
authors etc.

Once I've selected the "Course" that seems to be the best one, I'm notified that
there are few required major points to master - such as the basics of
the asynchronous programming - I try to find the best way to master those major points too.

Once I'm done I can go to my task list.

It shows a nice list of knowledge points - those I can jump into straight away -
I've already mastered all their dependencies. Ok, let's jump into one and start
doing my lectures and exercises.

Whoow, there's a surprise. The creator of node.js lecture has created a nice 2D
map of all terms I'd have to learn in the "course" - that looks lust like maps
in Super Mario, nice :)

Next day I'm doing few lectures while commuting to my job - those are short
5-minute videos so I can easily pause at any time I want and continue later.

Finally after few weeks there's an exam - I take a deep breath and open a web page
with the test. It shows few questions and live code editor box. Each question
does validate my code and checks if it's ok. I'm answering the questions one by one.
Finally I've done all of them. My answers are sent to the final validation.
Few minutes later I receive a message that I've passed, hurray !!!

I quickly grab the link to my accomplishment and post it onto my LinkedIn account.
I did master *node.js* since I want to change my job soon. Such accomplishment badge will be
helpful. LinkedIn servers quickly verify all signatures of the accomplishment -
soon my profile gets another nice icon in my badges page.

Few days later I got the first interview invitation. Recruiter was looking for
people who mastered *node.js* and knows some *C++* - I mastered *C++* half a year ago,
that was a good move.

Recruiter knows exactly what's my level of the knowledge. There are few topics
I'm still not yet good enough at but I already got the list from the job
description so I know what to prepare next.

Some time has passed and I got my new job. After the introduction, I start getting used to
some internal tools created by the company. Of course they have their internal K-Dag
server so even though the topics I master here won't be made public, I already know
how to operate there and the system knows what I've mastered so far.

Few months later I'm sent to a training session. I know it's pretty expensive so I
could never afford that by myself. Bu the company sees a big potential in me so they
believe it's worth it. Before starting the course I've sent my profile to the trainer.
He was well prepared - he knew exactly what the background
of our group was so he tuned the material to our current knowledge. Once we've
completed the course, he also signed all certificates for us - my profile has been
updated once again, new achievements unlocked on my LinkedIn account :)

The idea is here, what's next?
------------------------------

I don't know how about you but I'm sold on the idea and would like to implement it
one day. It could be made fully decentralized and distributed, could be shaped in
a way to make the knowledge more approachable and accessible.
It could also have many implementations as long as the core is strong and well
thought off. There are also many ways to monetize K-DAG so why not try doing it?

So my plan now is to create some trivial prototype . You can expect some blog posts
about it soon.
