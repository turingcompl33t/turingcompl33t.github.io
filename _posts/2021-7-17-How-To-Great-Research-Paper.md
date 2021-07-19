---
layout: post
title: "How to Write a Great Research Paper"
---

I recently watched the presentation [How to Write a Great Research Paper](https://www.youtube.com/watch?v=VK51E3gHENc) by Simon Peyton Jones. I am already a huge fan of Simon's, both his work and his speaking appearances, and this presentation still managed to exceed my expectations, so much so that I feel the need to write this recapitulation and response. While I endeavor to cover some of Simon's main points in this post, it is impossible to convey the infectious passion and charisma with which he delivers them. If you have not done so already, I implore you to watch [the original](https://www.youtube.com/watch?v=VK51E3gHENc) for yourself.

In addition to summarizing the presentation, I also provide some commentary of my own. I hope that this might add perspective - that of a truly-novice researcher - that others might find informative or at the very least amusing. For context, I have never written a proper research paper, let alone a great one. I am, however, in the process of conducting research (see [here](https://noise.page/) and [here](https://github.com/turingcompl33t/udj-jit) for specifics) in partial fulfillment of the requirements of my current academic program at Carnegie Mellon University. While I have been working on various [implementation aspects](https://github.com/cmu-db/noisepage/pull/1510) of this research for the better part of an academic term, I only recently began focusing more of my attention on reading the corpus of related work and writing drafts of my own. For this reason, I thought it prudent to begin thinking about how to communicate my research effectively.

With that motivation and context out of the way, let's jump into Simon Peyton Jones' seven suggestions for writing a great research paper.

### 1. Start Writing

Simon (roughly) structures his presentation around the authorship process. Accordingly, he begins by orienting us to where writing fits into the larger research process. Simon recommends that we treat writing our paper as an integral part of the research itself, and not merely as the final step in which we communicate results - the side effect of an otherwise purely-functional process (forgive me, FP gods). To make this idea more concreate, we might imagine that the "standard" procedure for conducting research looks something like:
1. Get Idea
2. Do Research
3. Write Paper

In contrast, Simon's advocates the reordering:
1. Get Idea
2. Write Paper
3. Do Research

Instead of waiting until after the research is complete to begin writing the paper, we begin writing the paper before we even begin conducting the research. This ordering might at first appear counterintuitive ("what am I going to write about if I haven't _done_ anything yet?") but adopting this strategy has some immediate implications that benefit us in the long run:
- **It forces us to clarify our idea.** When an idea exists only in your head, it is much easier to avoid consideration of its complexities or hide an imprecise understanding. Articulating the idea in writing exposes these weaknesses.
- **It helps direct the research we will conduct.** When we only begin to write the paper after the research is complete, we constrain ourselves to writing the paper that is enabled by the research. If we begin the research process with the paper, we can instead write _the paper that could be_ - the best version of the paper that addresses our idea - and conduct the research necessary to realize it.

This is one of the suggestions from Simon's presentation that I regret not considering sooner. As mentioned above, I have already completed some of the core implementation aspects of my research, and am only now beginning to think about writing and how I want the paper to look. While the two points above are certainly relevant to my situation as well, I feel another detrimental aspect of the "traditional approach" arises in the context of related works. In beginning my research, I read the papers that I thought directly relevant and stopped there. I have since read a significantly-larger subset of the relevant papers in my field, and I find my conception of how my research fits into this broader picture changing as a result. Ultimately I don't think this will kill the paper, but it certainly introduces additional work - work that may have been elided had I heeded Simon's first suggestion. 

### 2. Know What Your Idea Is

A paper is not a way to get "research point" but rather is a mechanism for conveying an idea. An effective paper is like a mind-virus - it takes over the mental space of the people who read it.

Papers are far more durable than programs. They convey ideas. Think Mozart - we still use his ideas in concert halls today!

Fallacy: thinking that your idea is insigificant or not worth sharing. Even if your idea is "weedy" and insigificant, write it anyway. Not much is loss if this turns out to be true, and if you are wrong, you have an important idea on your hands!

"People don't have good ideas, rather we discover that they are good ideas after the fact."

The paper needs one, clear, sharp idea. If you have many ideas, write many papers.

### 3. Tell A Story

Now that we know _what_ our idea is, we need to figure out _how_ best to communicate it to the world. Simon recommends imagining the process of explaining your idea at a whiteboard, and comes up with the following narrative structure:
- Here is a problem
- It is an interesting problem
- It is an unsolved problem
- Here is my idea
- My idea works
- Here is how my idea compares to others' approaches

Anyone who has done any paper-reading should recognize this as the general structure of most research work. The critical insight that I took away from this section of Simon's presentation is that **there is a reason papers are structured this way.** That is, the typical or expected sections of a research paper are not just empty boxes with an attached label, waiting for us to fill them with content. Instead, **these are the sections that naturally emerge when we tell a story** which is ultimately what we want to do.

Despite being very new to the process of writing research papers, I already find that I frequently get bogged-down in filling in the content that is expected of me in each section of the paper. This is the wrong way to think about the problem; indeed, it might even be the _opposite_ of the right way! The content in each section of the paper should be a consequence of the story we want to tell - it is not the goal in and of itself. This is a lesson that I learned again and again in English / Language Arts classes throughout high school and my undergraduate degree, but for some reason it never occurred to me that it should be applied to technical writing the same way it is applied to any other prose.

### 4. Nail Your Contributions to the Mast

We have identified the narrative that drives our paper as well as the high-level structure that this narrative will take. Now we turn our attention to the content of the paper itself, beginning with the introduction.

Simon recommends a minimal introduction that consists of just the following two elements:
- State the problem
- Identify your contributions

There are a number of good reasons for keeping the introduction concise, the most important being that the time and attention of our readers is limited. The introduction may be the only section of our paper that someone reads, and at the very least it is our chance to capture readers' attention.

Simon's advice on the best way to do this is to lead with a specific example. It is natural for us to want to provide sufficient context for the problem prior to proposing our solution, but more often than not this context hurts our introduction more than it helps it. We wander through several paragraphs of background information and motivation, and by the time we reach our idea, we have lost our reader's focus. The alternative is to immediately identify our little corner of the problem space and attack it directly. To use Simon's analogy, we should not waste our precious space in the introduction describing the entire mountain, instead we should quickly focus in on the molehill that our idea addresses.

To make this idea concrete, let's look at a specific example. The passage below is taken from the introductory section of a preliminary report on my research:

> Structured query language (SQL) is a cornerstone of data storage and retrieval. The language, despite being developed nearly fifty years ago, represents one of the last remaining commonalities among most modern database management systems (DBMS). There are a number of reasons for SQL's continued popularity, the most salient being its high-level declarative syntax that makes it both easy to learn and amenable to optimization. 
> 
> Despite its success, however, SQL is not without its drawbacks...

Reading this now is somewhat embarassing, but also amusing; in this passage, I write _exactly_ the style of introduction that Simon tells us to avoid! While it might sound nice (or at least it did to me when I wrote it) this introduction does no useful work. Readers new to the field will likely be confused (e.g. why does SQL's declarative syntax make it amenable to optimization?) while readers that are familiar with database systems don't need another lecture on the virtues of SQL.

Revisiting this introduction and applying some of Simon's recommendations, I might now write something like:

> Consider the SQL queries below:
> ```SQL
> -- Table test
> -- -----------
> -- | x (INT) |
> -- -----------
>
> SELECT x, 1 FROM test;
>
> SELECT x, return_one() FROM test;
> ```
> While these queries produce equivalent results, the latter query may take up to 1000x longer to execute than the former in some database systems. This surprising disparity in runtime is a result of the use of the user-defined function (UDF) `return_one()` in the latter query. We present an new approach to efficient execution of UDFs in relational database systems that reduces the negative performance impact they impose.

This revised introduction is far from perfect (and the result about execution time is bogus, so don't cite me on this) but it gets the point across. I avoid wasting all that time up front talking about SQL and get right to the crux of the problem: user-defined functions are slow, and we're going to fix it.

One other recommendation that Simon makes regarding the content in our introductions is to **make contributions explcit and refutable**. By making our contributions explicit, we make it impossible for the reader to misunderstand where prior work stops and our proposed approach begins. An easy way to make contributions explicit is by itemizing them with bullets in the body of our introduction. 

Ensuring that our contributions are refuatable requires slightly more throught. By _refutable_, Simon means that our contributions must be things that _could be_ false - promises on which we fail to deliver. Again, I'll use an example from my own research to illustrate this point. Consider the following list of contributions:

> - We describe the performance bottlenecks of user-defined function execution.
> - We describe a system for UDF execution via JIT-compilation.

Now, compare those with this revised list:

> - With performance profiling results from representative analytical workloads, we show that the overhead imposed by context-switches between relational and imperative execution engines accounts for 80% of the performance degredation introduced by UDFs.
> - We implement support for JIT-compilation of user-defined functions in the NoisePage DBMS. Our results show that this new execution method improves overall query runtime by up to two orders of magnitude relative to an interpreted baseline.

The former list contains horrible, strawman examples that I never actually included in a paper or report, but they certainly look like something I _might_ have written had I not considered the important of refutability.

### 5. Leave Related Work for Later

Problem 1: You have not gotten to your idea yet, so your compressed description of alternative approaches is incomprehensible.
Problem 2: Describing alternative approaches gets between the reader and your idea.

We are taking the reader to our idea via the most direct path.

Credit is like love, not money. It is not a zero-sum game. You do not need to make others' ideas look bad to make yours look good or worthy of consideration.

Acknowledge weaknesses in your approach. It is very rare in computer science to come up with a new idea that dominates all existing approaches along all dimensions.

### 6. Put Your Readers First

Don't recapitulate your entire journey for your readers. You were wandering blind while working on initial iterations of your idea, and while it may be tempting to walk readers through this as well, it will put them to sleep and waste their time.

I am definitely prone to making this mistake. I am tempted to take my readers by the hand and lead them through the twists and turns of the research maze so that they might appreciate how clever I am to have made it out the other side. But this is not the point.

This highlights an even more fundamental principle that might not occur to novice researchers (and may be forgotten by those with more experience): scientific research is not about _you_. I want to see my name on the authors line as much as the next guy.

Conveying the intuition of the payload of your idea is primary, not a secondary concern.

Introduce the concepts with examples, and only after that get to the general case with rigorous formulation (if applicable).

Explain it as if you were speaking to someone at a whiteboard.

### 7. Listen to Your Readers

Get your paper read by as many people as possible - guinea pigs.

Experts are an important audience. But non-experts are an important audience too.

Use your guinea pigs sparingly. Once someone has read your paper once, they will never read it the same way again if you ask them read a revision (likely to put as much effort in).

Explain carefully to your guinea pigs what you want from their review. You don't care about spelling and grammar mistakes. You do care about where they lose understanding in the paper. Make this desire explicit!

Treat every review like gold dust. Be (truly) grateful for criticism. You can use this to make your work better. Also respect the fact that the reviewer, regardless of the content of their review, have given you the gift of their time.

### Conclusion



### References

- [How to Write A Great Research Paper](https://www.youtube.com/watch?v=VK51E3gHENc) talk by Simon Peyton Jones.
