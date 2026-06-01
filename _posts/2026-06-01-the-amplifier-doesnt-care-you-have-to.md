---
layout: post
title: "The Amplifier Doesn't Care. You Have to."
date: 2026-06-01
author: "Ihor Katkov & Ivan Pachesnyi"
description: "AI amplifies capability, incentives, and harm. The question is what we point it at."
image: "/assets/img/ai-amplifier-commons.webp"
tags: [ai, ai-agents, software-engineering, engineering-leadership, open-source]
---

{% include figure.liquid path="assets/img/ai-amplifier-commons.webp" class="img-fluid rounded z-depth-1" alt="Tower illustration for The Amplifier Doesn't Care" caption="<em>The problem was never the tower. It was what people thought they were building together.</em>" zoomable=true %}

*Originally posted on [Substack](https://ihorkatkov.substack.com/p/the-amplifier-doesnt-care-you-have).*

Coding is solved. Coding is nowhere near solved. AI will replace all engineers within 5 years. Amazon stated that senior engineers [review every line](https://www.techradar.com/pro/amazon-is-making-even-senior-engineers-get-code-signed-off-following-multiple-recent-outages) of AI-generated code because the AI kept breaking production. AI is the most important technology since the Internet. AI is a bubble, just like blockchain and the metaverse. The software sector lost $800 billion in market capitalization in three months. AI companies are raising money faster than at any point in history. Programmers have never been more productive. Programmers are losing productivity. [Everyone is hiring](https://www.lennysnewsletter.com/p/state-of-the-product-job-market-in-ee9?hide_intro_popup=true)? [Nobody is hiring](https://www.nytimes.com/2026/03/24/business/economy/college-graduates-job-market-hiring.html). You will be fine. You are so over.

A year ago, I would never have believed what software engineering looks like today. Coding agents or even swarms of them, meta harnesses, and long-running fully automated sessions are now becoming the new normal.

Yet, I remember the first time I experienced my “Oh, f*ck” moment, when my coding agent implemented a whole rate-limiting library in the Elixir ecosystem I had always wanted to do.

What should I do with it? Should I tell my manager and my team? How will it impact my day-to-day life? Should I automate the manual workflow I do every day, and will it endanger my job security? What happens to my job, my specialty, and the value of the things I spent years learning?

This is what the AI transition looks like from the inside. Not the headlines. A specific person, in a specific office, with a specific decision to make about whether to be honest about what just happened.

In this article, we, **Ihor** and **Ivan**, two engineers who have spent more than a decade building software systems, write down what we think is actually happening in the industry, what worries us, and what we believe engineers should do next.

Six claims. We will defend each one.

### AI is an amplifier of capability built on a commons.

The purpose of a system is what it does, a claim famously attributed to Stafford Beer. What AI does, stripped of the marketing, is amplify individual capability by drawing on a collective body of knowledge. That is what the algorithm does mechanically. The [internal mechanism is more structured](https://dnhkng.github.io/posts/rys/) than the popular “it just predicts the next word” summary suggests, and we have yet to fathom its inner workings.

The end result is that the mechanism amplifies whatever capability you point it at. For instance, a research team from France [transcribed 32,000 medieval manuscripts in four months](https://www.medievalists.net/2026/01/32000-medieval-manuscripts-transcribed-using-ai/), work that would have taken human scholars decades, while Romania’s 2024 presidential election was annulled after [AI-generated deepfakes interfered with the results](https://www.cigionline.org/articles/then-and-now-how-does-ai-electoral-interference-compare-in-2025/).

The same process that transcribes manuscripts and helps you lose weight also fabricates election results and poisons information systems. The mechanism does not care; you have to. The minimum it takes is judgment to evaluate what comes out, and enough basic decency to consider who it affects besides you.

## The training data was taken. The response has to be more than retreat.

Every Stack Overflow answer you’ve written, every public GitHub repo, every technical blog post, every forum explanation where you helped a stranger debug their code is in the training data. The legal and ethical frameworks for training data are being built right now, too slowly for those already affected.

That said, the collectively elaborated knowledge base now exists, assembled without consent and available to everyone. Nevertheless, it gives something very substantial. It let us build things that were not possible before, explore ideas from angles we would not have noticed, and judge our own work against a broader context than any single person could hold in their head.

Carefully crafted text by hand, systems, and judgment remain more valuable than what models are producing on their own. The model is drawing on your work. You can draw on it back, and what you make with it can be worth more than what was taken.

The temptation of demonstrative rejection is strong: refusing to engage with AI publicly, as a stance. It is understandable. It is also strategically poor, for tyranny always needs its loyal opposition, and fighting the tool is exactly what puts up an act of disagreement with no meaningful change to the status quo. Simply because the tool in question will keep getting cheaper and more capable. Fighting the decision means engaging in the same kind of struggle that every profession has faced when new technology made old workflows cheaper to automate. That fight is between people, about how the value of work is distributed.

Demonstrative rejection does not teach you to fight what you hate. Whoever understands the tool will shape the way creative work is done.

## Hoarding the workflow is a coordination failure dressed as cleverness.

This is the position that is not usually stated out loud. The executive who uses AI as the stated reason for layoffs that are really about cost discipline; the engineer who found a productive AI workflow and now does two hours of work in an eight-hour day, keeping the method private; the company or country concentrating AI capabilities without intending to distribute the benefit; each of them is extracting value from the transition while it lasts.

This approach works in the short term. In the medium term, it breaks, and the mechanism is the same at every scale. You cannot strip a system of its experienced components and expect it to keep functioning and growing.

The engineer hoarding a workflow faces a specific version of this. When management discovers that AI increases team productivity, they will expect more from everyone. When only one person knows how to leverage AI for greater productivity, that person has leverage for a period of time. But when the whole team knows, they can set terms, standards, and expectations together. Five times the work for the same pay, or the same work at a higher level of quality, with time left for harder problems. This is a coordination problem, the same shape as every labor coordination problem in history.

The largest-scale version is geopolitical. A world where AI capabilities are concentrated in a few companies or countries is fragile. Monocultures do not adapt well. The internet was built by many people in many countries, contributing different pieces. The most resilient systems humanity has built have been the distributed ones.

What to do depends on where you are. If your company lays off people to improve AI efficiency and increase its market valuation, and you are part of it, there is nothing you can do. Sometimes it makes sense to let the system fail. Brace yourself and move on.

If you’re worried about your position in the company, lead AI adoption. Become the expert and the go-to person. When you find a way to make yourself five times faster, share it with the team and make it visible for everyone in the company. You get better visibility, but as a team, you can also negotiate what it means for you and for your working conditions.

When you’re choosing tools and platforms, choose the ones that contribute to society. Open source, shared tooling, distributed infrastructure. You can’t fix geopolitics from your desk, but you can decide whether your work adds to the commons.

### Engineers are not being replaced. The definition of engineering is being rewritten

Amazon, a company too big to fail if there ever was one, tried to cut the biggest corner known to humanity: humanity itself. It [laid off tens of thousands of engineers citing AI efficiency](https://www.ciodive.com/news/aws-layoffs-cloud-AI-andy-jassy/804046/), [deployed 21,000 AI agents](https://awesomeagents.ai/news/amazon-ai-code-review-outages-senior-approval/), and then [suffered a string of production outages](https://fortune.com/2026/03/12/amazon-retail-site-outages-ai-agent-inaccurate-advice/) when the institutional knowledge it discarded turned out to be load-bearing. This doesn’t sound like a successful replacement at all.

The tempting management story is simple: if AI writes code, fewer engineers are needed. But production systems do not run on code alone. They run on context, ownership, incident memory, architectural judgment, and the thousand small decisions that never make it into documentation.

Replacing humans with AI is not a straightforward operation, and no one has pulled it off at scale yet. What Amazon discovered is that the skills that matter, system thinking, institutional knowledge, judgment about what can go wrong, are exactly the ones AI does not have. Those skills were always valuable, just less visible when writing the code was the hard part. Now that implementation is getting cheaper, they’re what’s left standing.

The same thing happened with Toy Story in 1995, which felt like a revolution and, within a few years, became the default expectation. We’ve already mentioned cases where one of us built a production ML pipeline in a month with no machine learning background, because he knows system design, and the AI handled implementation. The other ran design, DevOps, backend, and frontend on a product that has served thousands of users for years. And again, a year ago, either of those would have required a team. The tools made each engineer capable of more, provided that the engineer knows what to build and how systems fit together.

Goldman Sachs’ analysts reached the same conclusion on the capital markets side in their March 2026 report: AI is software, and rather than eating the software market, it is likely expanding it. The risk is not that engineering disappears but that engineers who do not adapt become incumbents in a market that moved past them.

### Juniors are still needed. The definition of junior work has changed.

In 2025, Salesforce CEO Marc Benioff announced the company would hire no new engineers, citing AI agents. [A professor writing in Science admitted](https://www.science.org/content/article/why-i-may-hire-ai-instead-graduate-student) they were tempted to give their research tasks to AI rather than recruit a graduate student, because AI delivers immediate returns while a junior’s value emerges slowly. The logic is the same in both cases: if the tool can handle the entry-level work, why invest in someone who needs months to ramp up?

The logic is correct regarding the old tasks, but wrong regarding the conclusion. The tasks that defined junior work are now handled by agents. What follows is that the definition of junior work has changed, not that juniors are unnecessary. If you stop hiring juniors, you stop producing the next generation of seniors. The senior engineers that Amazon now requires for code review were all juniors once. Cut the entry point, and you eventually run out of the people whose oversight you depend on.

So juniors are needed, but a different kind. The old junior learned by doing simple, low-risk tasks: changing a button, adding a database field, writing a filter. That work was automated, and anyway, candidly, was rarely enjoyed, if at all. What a useful junior looks like now is someone who understands how systems fit together and can direct AI tools to build the parts, someone whose value is in knowing what to build rather than in typing it out. The market is harder and lower-paid than the one you were promised when you started studying, that’s true. But that is not a reason to give up. It is a reason to be deliberate about what you learn.

This topic is too large and too important for a subsection. We intend to dedicate a separate piece to what juniors should actually learn now, what companies should change about hiring and onboarding, and how to build judgment in an environment that constantly tempts you to outsource it.

## The hype is real in its consequences, whether or not it is real in its claims

The layoffs we mentioned earlier were not imaginary, and ignoring them didn’t make them go away. Words have power, specifically the power to elicit action. When a CEO tells a board that AI will make 30% of the workforce unnecessary, it does not matter whether the claim is technically precise. What matters is that the board approves the headcount reduction.

Whether AI lives up to its claims in 2025 and 2026 remains an open question. That it has already changed hiring decisions, compensation structures, market valuations, and the daily work of millions of people is not open. It already happened.

The useful version of skepticism is not dismissal. It is scrutiny. The most valuable people in this moment are neither true believers nor deniers. They are the ones who can tell the difference between what works and what is marketing. That requires looking closely, not looking away.

### So what is the work?

The job is no longer about writing code. The best code was always the code you did not have to write, because every line is a liability that must be maintained, debugged, secured, and eventually replaced. AI accelerates this principle. The question that matters is “should this system exist, what should it do, and how should it fit together?” That is system design. It has always been the core of engineering, but for decades, it was buried under the mechanical difficulty of implementation. Now that implementation is getting cheaper, design surfaces as the thing that actually matters.

And the work remains collaborative. Individual output is growing, but the systems being built still exceed what one person can hold in their head. Architecture, product strategy, user needs, cross-team alignment, all these remain team problems. The open-source principle holds: people built things together that none of them could have built alone. That does not change because individual contributions have gotten larger.

We do not have complete answers. The field changes monthly, and anything we claim to know for certain will probably need revision by next quarter. But we can say what we observe: the transformation is real, it is already underway, and the people who will shape it are the ones who engage rather than retreat. The tool is powerful. The decisions about how to use it are made by people. The work is still there. It still needs people.

Whatever brought you to this article, whether it was fear, curiosity, frustration, or just the inability to sleep, you have your own reasons for reading. We have ours for writing. They come down to the same thing: there is work to do, the terms of it are changing, and it is better to have a hand in shaping those terms than to watch someone else set them. If you are reading this at 3 AM with bloodshot eyes, overcaffeinated, in the grip of an existential crisis about whether you will have a career in two years: slow down. Close the browser. Go to sleep. The Earth will still be spinning in the morning. Then learn the tool. Question the incentives. Share what works. Protect the commons. Build with judgment.

The amplifier does not care. You Have to.

— Ihor Katkov & Ivan Pachesnyi.
