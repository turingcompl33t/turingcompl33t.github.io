---
layout: post
title: "How to Write a Great Research Paper"
---

I recently watched the presentation [How to Write a Great Research Paper](https://www.youtube.com/watch?v=VK51E3gHENc) by Simon Peyton Jones. I am already a huge fan of Simon's, both his work and his public speaking, and this presentation still managed to exceed my expectations. While I endeavor to cover some of Simon's main points in this post, it is impossible for me to convey the infectious passion and charisma with which he delivers them. I implore you to watch [the original](https://www.youtube.com/watch?v=VK51E3gHENc) for yourself.

In addition to summarizing the presentation, I also provide some commentary of my own. I hope that this might add perspective - that of a truly-novice researcher - that others might find informative or at the very least amusing. For context, I have never written a "proper" research paper, let alone a great one. I am, however, in the process of conducting database systems research associated with [the NoisePage project](https://noise.page/) (some additional resources are [here](https://github.com/turingcompl33t/udj-jit)) in partial fulfillment of the requirements of my current academic program at Carnegie Mellon University. While I have been working on various [implementation aspects](https://github.com/cmu-db/noisepage/pull/1510) of this research for the better part of an academic term, I only recently began focusing more of my attention on reading the corpus of related work and writing drafts of my own. For this reason, I thought it prudent to begin thinking about how to communicate my research effectively.

With that motivation and context out of the way, let's jump into Simon Peyton Jones' seven suggestions for writing a great research paper.

### 1. Start Writing

Simon (roughly) structures his presentation around the authorship process. Accordingly, he begins by stating that we must treat writing our paper as an integral part of the research itself, not merely as the final step in which we communicate results. To make this idea more concreate, the typical procedure for conducting research might look something like:
1. Get Idea
2. Do Research
3. Write Paper

In contrast, Simon's advocates the reordering:
1. Get Idea
2. Write Paper
3. Do Research

Instead of waiting until after the research is complete to begin writing the paper, we begin writing the paper before we even begin conducting the research. This ordering might at first appear counterintuitive ("what am I going to write about if I haven't _done_ anything yet?") but adopting this strategy has some immediate implications that benefit us in the long run:
- **It forces us to clarify our idea.** When an idea only exists in your head, it is much easier to avoid its more complex aspects or hide an imprecise understanding of it. Articulating the idea in writing helps uncover these weaknesses.
- **It helps direct the research we will conduct.** When we only begin to write the paper after the research is complete, we constrain ourselves to writing the paper that is enabled by the research. If we begin the research process with the paper, we can instead write _the paper that could be_ - the best version of the paper that addresses our idea - and conduct the research necessary to realize it.

This is one of the suggestions from Simon's presentation that I regret not considering sooner. As mentioned above, I have already completed some of the core implementation aspects of my research, and am only now beginning to think about writing and how I want the paper to look. While the two points above are certainly relevant to my situation as well, I feel the most detrimental aspect of the "traditional approach" comes in the area of related work. In beginning my research, I read the papers that I thought directly related to it and stopped there. I have since read a significantly-larger subset of the relevant papers in my field, and I find that my conception of how my research "fits" into this broader picture changing slightly. Ultimately I don't think this will kill the paper, but it certainly introduces additional work - work that may have been elided had I heeded Simon's first suggestion. 

### 2. Know What Your Idea Is

A paper is not a way to get "research point" but rather is a mechanism for conveying an idea. An effective paper is like a mind-virus - it takes over the mental space of the people who read it.

Papers are far more durable than programs. They convey ideas. Think Mozart - we still use his ideas in concert halls today!

Fallacy: thinking that your idea is insigificant or not worth sharing. Even if your idea is "weedy" and insigificant, write it anyway. Not much is loss if this turns out to be true, and if you are wrong, you have an important idea on your hands!

"People don't have good ideas, rather we discover that they are good ideas after the fact."

The paper needs one, clear, sharp idea. If you have many ideas, write many papers.

### 3. Tell A Story

Imagine explaining your idea at a whiteboard:
- Here is a problem
- It is an interesting problem
- It is an unsolved problem
- Here is my idea
- My idea works
- Here is how my idea compares to others' approaches

### 4. Nail Your Contributions to the Mast

- State the problem
- Identify your contributions
- That's it!

Simon's advice: use a specific example.

Molehills not mountains. Don't get bogged down in too much framing or context for the problem (the whole moutain). Instead, slice off your molehill and attack it directly.

Explicitly state your contributions. Use bullets. Your contributions should be _refutable_ - they should be things that could be false, that you fail to deliver on! 

Skip the section with "the rest of this paper is structured as follows..." Everybody skips this! Instead, use forward references to the rest of your paper throughout the introduction.

Use your introduction as a dead-code-elimination pass on your paper. If you have a section in your paper that you do not reference in your introduction, it is likely not that important and perhaps it can be removed.

### 5. Leave Related Work for Later

Problem 1: You have not gotten to your idea yet, so your compressed description of alternative approaches is incomprehensible.
Problem 2: Describing alternative approaches gets between the reader and your idea.

We are taking the reader to our idea via the most direct path.

Credit is like love, not money. It is not a zero-sum game. You do not need to make others' ideas look bad to make yours look good or worthy of consideration.

Acknowledge weaknesses in your approach. It is very rare in computer science to come up with a new idea that dominates all existing approaches along all dimensions.

### 6. Put Your Readers First

Don't recapitulate your entire journey for your readers. You were wandering blind while working on initial iterations of your idea, and while it may be tempting to walk readers through this as well, it will put them to sleep and waste their time.

Conveying the intuition of the payload of your idea is primary, not a secondary concern.

Introduce the concepts with examples, and only after that get to the general case with rigorous formulation (if applicable).

Explain it as if you were speaking to someone at a whiteboard.

### 7. Listen to Your Readers

Get your paper read by as many people as possible - guinea pigs.

Experts are an important audience. But non-experts are an important audience too.

Use your guinea pigs sparingly. Once someone has read your paper once, they will never read it the same way again if you ask them read a revision (likely to put as much effort in).

Explain carefully to your guinea pigs what you want from their review. You don't care about spelling and grammar mistakes. You do care about where they lose understanding in the paper. Make this desire explicit!

Treat every review like gold dust. Be (truly) grateful for criticism. You can use this to make your work better. Also respect the fact that the reviewer, regardless of the content of their review, have given you the gift of their time.

### References

- [How to Write A Great Research Paper](https://www.youtube.com/watch?v=VK51E3gHENc) talk by Simon Peyton Jones.
