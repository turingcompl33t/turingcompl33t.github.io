---
layout: post
title: "How to Write a Great Research Paper"
---

I recently watched the presentation [How to Write a Great Research Paper](https://www.youtube.com/watch?v=VK51E3gHENc) by Simon Peyton Jones. In this post, I summarize some of my primary takeaways from this presentation and augment this with some examples from my own research experience.

For those unfamiliar with Simon Peyton Jones, he is best known as one of the minds behind [Haskell](https://www.haskell.org/), a purely-functional programming language that lends itself to [concise, beautiful implementations](https://wiki.haskell.org/Introduction#Quicksort_in_Haskell). Aside from his many contributions to the Glasgow Haskell Compiler, he is also a [prolific researcher](https://scholar.google.fr/citations?user=QsX7G-cAAAAJ&hl=en). so when he speaks on the subject of effective research, we would do best to listen.

While I endeavor to cover some of Simon's main points in this post, it is impossible to convey the infectious passion and charisma with which he delivers them. If you have not done so already, I implore you to watch [the original](https://www.youtube.com/watch?v=VK51E3gHENc) for yourself.

In addition to summarizing the presentation, I also provide some commentary of my own. I hope that this might add perspective - that of a truly-novice researcher - that others might find informative or at the very least amusing. 

For context, I have never written a proper research paper, let alone a great one. I am, however, in the process of conducting research (see [here](https://noise.page/) and [here](https://github.com/turingcompl33t/udj-jit) for specifics) in partial fulfillment of the requirements of my current academic program at Carnegie Mellon University. While I have been working on various [implementation aspects](https://github.com/cmu-db/noisepage/pull/1510) of this research for the better part of an academic term, I only recently began focusing more of my attention on reading the corpus of related work and writing drafts of my own. For this reason, I thought it prudent to begin thinking about how to communicate my research effectively.

Note that both above and in the remainder of this post I refer to Professor Peyton Jones simply as "Simon." This is not to suggest that we are on a familiar, first-name basis, nor is it out of a lack of respect for his credentials and the titles they imply. In the first draft of this post I used his more decorous identifier, but I found in revisions that substituting this with his first name makes the post more pleasant to read. If you are reading this, Professor, forgive me!

With that motivation and context out of the way, let's jump into Simon's seven suggestions for writing a great research paper.

### 1. Start Writing

Simon (roughly) structures his presentation around the authorship process. Accordingly, he begins by orienting us to where writing fits into the larger research process. Simon recommends that we **treat writing our paper as an integral part of the research itself**, and not merely as the final step in which we communicate results - the side effect of an otherwise purely-functional process (forgive me, FP gods). To make this idea more concreate, we might imagine that the "standard" procedure for conducting research looks something like:
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

This is one of the suggestions from Simon's presentation that I regret not considering sooner. As mentioned above, I have already completed some of the core implementation aspects of my research, and am only now beginning to think about writing and how I want the paper to look. 

While the two points above are certainly relevant to my situation as well, I feel another detrimental aspect of the "traditional approach" arises in the context of related works. In beginning my research, I read the papers that I thought directly relevant and stopped there. I have since read a significantly-larger subset of the relevant papers in my field, and I find my conception of how my research fits into this broader picture changing as a result. 

Ultimately I don't think this will kill the paper, but it certainly introduces additional work - work that may have been elided had I heeded Simon's first suggestion. 

### 2. Know What Your Idea Is

We've established that starting to write early in the research process is of paramount importance, but we can't just start spewing words into a Latex document and hope that a groundbreaking result emerges. We need an idea to organize and direct our efforts. Accordingly, Simon's second suggestion is aimed at describing how such an idea should look.

Simon presents a number of concrete suggestions here, but they all flow naturally from the observation that **an effective idea is like a mind-virus** - it invades and inhabits the minds of those who read it, whether they are conscious of this occurrence or not. This analogy might sound a bit dark (perhaps even malicious) but consider some of the properties that it implies our idea must exhibit:
- It must be simple (this is NOT to say that it is "easy")
- It must be amenable to concise expression
- It must admit of an intuitive understanding
- It must solve a problem, and do so elegantly

Think of the last compelling technical idea you heard. Did it have any of these properties? Would you still be able to communicate it to someone else with high fidelity?

Simon cites Mozart as an example of an author of ideas that fit these criteria; we still read his work to this day in concert halls all over the world. 

For me, the example that comes most-readily to mind is the approach to writing a query compiler through generative programming introduced in [this paper](https://arxiv.org/pdf/1612.05566.pdf) (and improved upon [here](https://www.cs.purdue.edu/homes/rompf/papers/tahboub-sigmod18.pdf)). It might not have the same widespread appeal as Mozart, but it is concise, it is distinct, and its elegance announces itself in my brain at least once a day.

Simon also touches on dealing with doubts regarding the significance of our idea. **even seemingly-inconsequential ideas are worth sharing**. After all, what are the tradeoffs here? If you are correct and your idea is unimportant, few people will read the paper, so the embarrassment blast radius will be small. However, if you are wrong, then the entire community benefits from your having taken this chance. Simon expresses this idea beautifully with the following quote:

> "People don't have good ideas, rather we discover that they are good ideas after the fact."

### 3. Tell A Story

Now that we know _what_ our idea is, we need to figure out _how_ best to communicate it to the world. Simon recommends imagining the process of explaining your idea at a whiteboard, and comes up with the following narrative structure:
- Here is a problem
- It is an interesting problem
- It is an unsolved problem
- Here is my idea
- My idea works
- Here is how my idea compares to others' approaches

Anyone who has done any paper-reading should recognize this as the general structure of most research work. The critical insight that I took away from this section of Simon's presentation is that **there is a reason papers are structured this way.** That is, the typical or expected sections of a research paper are not just empty boxes with an attached label, waiting for us to fill them with content. Instead, **these are the sections that naturally emerge when we tell a story** which is ultimately what we should strive to do.

Despite being very new to the process of writing research papers, I already find that I frequently get bogged-down in filling in the content that is expected of me in each section of the paper. This is the wrong way to think about the problem; indeed, it might even be the _opposite_ of the right way! 

The content in each section of the paper should be a consequence of the story we want to tell - it is not the goal in and of itself. This is a lesson that I learned again and again in English / Language Arts classes throughout high school and my undergraduate degree, but for some reason it never occurred to that it might be applicable to technical writing.

### 4. Nail Your Contributions to the Mast

We have identified the narrative that drives our paper as well as the high-level structure that this narrative will take. Now we turn our attention to the content of the paper itself, beginning with the introduction.

Simon recommends a minimal introduction that consists of just the following two elements:
- State the problem
- Identify your contributions

There are a number of good reasons for keeping the introduction concise, the most important being that the time and attention of our readers is limited. The introduction may be the only section of our paper that someone reads, and at the very least it is our first chance to capture readers' attention.

Simon's advice on the best way to do this is to lead with a specific example. It is natural for us to want to provide sufficient context for the problem prior to proposing our solution, but more often than not this context hurts our introduction more than it helps it. We wander through several paragraphs of background information and motivation, and by the time we reach our idea, we have lost our reader's focus. 

The alternative is to immediately identify our little corner of the problem space and attack it directly. To use Simon's analogy, we should not waste our precious space in the introduction describing the entire mountain, instead we should quickly focus in on the molehill that our idea addresses.

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
> SELECT x, (1) FROM test;
>
> SELECT x, return_one() FROM test;
> ```
> While these queries produce equivalent results, the latter query may take up to 1000x longer to execute than the former in some database systems. This surprising disparity in runtime is a result of the use of the user-defined function (UDF) `return_one()` in the latter query and the inefficiency with which it is evaluated. We present a novel approach to efficient execution of UDFs in relational database systems that reduces the negative performance impact they impose.

This revised introduction is far from perfect (and the result about execution time is bogus, so don't cite me on this) but it gets the point across. I avoid wasting all that time up front talking about SQL and get right to the crux of the problem: user-defined functions are slow, and we're going to fix it.

One other recommendation that Simon makes regarding the content in our introductions is to **make contributions explicit and refutable**. By making our contributions explicit, we make it impossible for the reader to misunderstand where prior work stops and our proposed approach begins. An easy way to make contributions explicit is by itemizing them with bullets in the body of our introduction. 

Ensuring that our contributions are refuatable requires slightly more throught. By _refutable_, Simon means that our contributions must be things that _could be_ false - promises on which we might fail to deliver. Again, I'll use an example from my own research to illustrate this point. Consider the following list of contributions:

> - We describe the performance bottlenecks of user-defined function execution.
> - We describe a system for UDF execution via JIT-compilation.

Now, compare those with this revised list:

> - With performance profiling results from representative analytical workloads, we show that the overhead imposed by context-switches between relational and imperative execution engines accounts for 80% of the performance degradation introduced by UDFs.
> - We implement support for JIT-compilation of user-defined functions in the NoisePage DBMS. Our results show that this new execution method improves overall query runtime by up to two orders of magnitude relative to an interpreted baseline.

The former list contains horrible, strawman examples that I never actually included in a paper or report, but they certainly look like something I _might_ have written had I not considered the importance of refutability.

### 5. Leave Related Work for Later

In a typical research paper, what section follows the introduction? In about 50% of the papers I have read, the answer is the section describing background and related work. Simon argues that this is a mistake, citing the following two problems as justification:
- Problem 1: At this point in the paper, you have not gotten to your idea yet, so your compressed description of alternative approaches is incomprehensible.
- Problem 2: Describing alternative approaches gets between the reader and your idea.

Based on my personal experience, I agree wholeheartedly with these observations. Indeed, most of the time that I encounter a related works section that comes before the actual contributions of the paper, I skim through it or skip over it entirely, and only on rare occasions do I circle back around to read it once I am prepared to do so. 

By saving the related work section until the end, we give ourselves the opportunity to **take the reader to our idea via the most direct path**. This maximizes the probability that they will have our idea in mind when they walk away from the paper.

Also on the topic of related work, Simon reminds us that **credit is like love, not money**. Technical research is not a zero-sum game, so we don't need to make others' ideas look bad to make our own look good or worthy of consideration. 

### 6. Put Your Readers First

We have covered the content in the introduction and related work, and finally we come to the heart of the paper - the description of our idea. Simon's advice here is simple yet powerful: **don't recapitulate your entire journey for your readers**. 

When we begin our research, the path forward may be unclear, and we may encounter many obstacles along the way. If you are anything like me, you might be tempted to subject your readers to a similar experience. 

We _can_ take our readers by the hand and lead them through the twists and turns of the research maze so that they might appreciate how clever we are to have emerged on the other side. But this is orthogonal to our true purpose, which is to communicate our idea. The journey may be interesting, but we should only include those aspects that are directly relevant to the end result. Save the gory details for a blog post or another alternative medium.

This highlights an even more fundamental principle that might not occur to novice researchers (and may be forgotten by those with more experience): **scientific research is not about you**. It is about innovating, advancing the state-of-the-art, and moving this entire human experiment forward (if only by an inch). 

Don't get me wrong, I want to see my name on the authors line as much as the next guy (especially in that hallowed first position). This can be a powerful motivator when working through some of the more difficult parts of the research process, but we must not allow this motivation to bleed through into the content of the paper itself. In doing so, we keep the focus where it belongs: the communication of our idea to the broader research community.

### 7. Listen to Your Readers

In his final recommendation, Simon addresses what is perhaps the most difficult part of the writing process: requesting and responding to feedback. 

The most important consideration to bear in mind here is to **use reviewers deliberately**. There are (at least) two actions we can take to ensure that we accomplish this:
- Enlist the help of reviewers sparingly. Once someone has read our paper once, they will never read it the same way again. They might skip over updated content or infer cohesion where it doesn't actually exist. However it manifests, the most productive reviews typically come from fresh eyes. We must therefore be judicious in deciding _when_ in the writing and revision process we should solicit reviews.
- Explain carefully to your readers what it is you want from their review. For instance, while spelling and grammar are important to get right, these are jobs for a word processor. From our human reviewers we typically want higher-level feedback, such as the precise points in the paper where their understanding falters. If we make the desire for reviews of this nature explicit, we can address these issues through subsequent conversations with our readers. Eliciting this type of feedback from a computer remains difficult (for now).

Finally, Simon encourages us to **treat every review like gold dust**. We must be truly grateful for the feedback we receive, regardless of its tenor. Whether they loved our work or hated it, our reviewer has given us the gift of their time - the one truly non-renewable resource we have to give. Furthermore, even if the reviewer's words are scathing, there is still likely at least one thing in them that we can use to make our work better.

I struggle mightily with this final suggestion. While I understand _intellectually_ that a less-than-positive review is (at least typically) not a personal attack, I can't help feeling embarrassed and defensive when I receive such feedback. I hope this is a reflex that will relax as I become accustomed to the experience.

### Conclusion

There you have it, seven actionable recommendations for writing better research papers. I hope you learned as much reading this post as I did writing it!

Thank you to Professor Simon Peyton Jones for sharing your suggestions and presenting them in such a compelling manner. I know at least one novice researcher who is benefiting greatly from them.

### References

- [How to Write A Great Research Paper](https://www.youtube.com/watch?v=VK51E3gHENc) by Simon Peyton Jones.
- [Building Efficient Query Engines in a High-Level Language](https://arxiv.org/pdf/1612.05566.pdf) by Amir Shaikhha, Yannis Klonatos, and Christoph Koch.
- [How to Architect a Query Compiler, Revisited](https://www.cs.purdue.edu/homes/rompf/papers/tahboub-sigmod18.pdf) by Ruby Y. Tahboub, Gregory M. Essertel, and Tiark Rompf.
