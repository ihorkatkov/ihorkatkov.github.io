---
layout: post
title: "From Skeptic to Believer: My Journey with the stdlib approach and AI agents"
date: 2025-04-01
author: "Ihor Katkov"
tags: [AI, software-engineering, productivity]
---

## Introduction

Programming is changing, and the pace has been getting even faster in recent years. LLMs are getting scary smart, and agents are being integrated everywhere. You need to try hard to avoid it. Although I'm an advanced user of modern LLM tools, I was highly skeptical about agent engineering. Oh boy, I was wrong. I tried the **stdlib** approach, which changed my view upside down.

## Background

Not long ago, I published [an article](https://ihorkatkov.github.io/blog/2025/ai-powered-software-development/) about my approach to using AI in day-to-day development. In a nutshell, I go to LLM, chat, and make specs. I then take them and save them into a file in the project folder while giving them to the LLM in the editor so that it can auto-complete better. I did believe in AI-augmented engineering but didn't believe in agents.

I tried using agent mode in Cursor AI, Windsurf, and Goose, but I was frustrated with the results because they were far from ideal. It made mistakes and failed to fix them, used a lot of API credits, and produced inconsistent results. I could not imagine how to integrate those approaches into my day-to-day workflow.

Recently, I stumbled onto [You're using Cursor AI incorrectly...](https://ghuntley.com/stdlib/) and decided to test it by publishing a slightly dusted idea of a circuit-breaking management library [brama](https://github.com/ihorkatkov/brama). I could deliver a fully working library written under my supervision, which is better than I would have done alone.

## The stdlib approach Explained

Geoffrey Huntley introduced a series of articles regarding the "stdlib" approach. He explains how he uses agents and Cursor Agent mode specifically. It's obvious if you think about it. Think about onboarding a new colleague engineer to your project. You give specs and documentation (if available ofc) of the project, give (or **teach**) engineering guidelines and approaches, and then give a small and highly detailed technical task to deliver. Why? Because when he provides the result, you can give feedback so that he can adjust and learn. And slowly, while results are consistently good, you give more advanced tasks.

That's precisely how stdlib works, but instead of a new human colleague, you have a new synthetic assistant. You write down documentation and [specifications](https://github.com/ihorkatkov/brama/blob/main/SPECS.md) and technical [rules](https://github.com/ihorkatkov/brama/tree/main/.cursor/rules), give him a task, and loop on itself to validate and rework if needed. As per the model, any thinking model will work, but Sonnet 3.7 is the best, as per my observation.

### The Framework:

**1. Write down rules/guidelines**

- It involves writing a list of comprehensive rules that could be applied to your project.
- Think about teaching a kid how to assemble a bike. You take a wrench and explain what it is and how to use it.
- For example, turn right to tighten a bolt, that's a pedal, and you put it here, etc.
- This is similar to what some companies have as an engineering rulebook.

**2. Provide rich context through maximally detailed specifications**

- Write down comprehensive documentation or specifications.
- This should be considered very carefully because it must involve **all the critical thinking you have**.
- Design flaws and mistakes could eliminate all the progress agents make while implementing the job.
- Include acceptance criteria, detailed explanations, context, examples, and clear goals.
- Imagine giving the job to another junior engineer and making what you expect crystal clear.

**3. Put rules and specs into the agent's context and loop it on itself**

- This is where execution happens.
- Load all the "how to execute" and "what to execute" in the context window.
- Strictly typed languages benefit the most, but others can also work.
- When you encounter any misbehavior, edit rules or specifications to teach the agent to do better.
- Currently, Claude 3.7 Sonnet works best for this job.
- Repeat until done.

## Case study: implementing brama

To test the approach, I decided to implement a circuit-breaking library, which I had considered long ago since I implemented the elemental one inside my current job. Here, [brama](https://github.com/ihorkatkov/brama) comes in.

Circuit breaking is a relatively straightforward pattern that helps our systems prevent cascading failures by temporarily blocking access to faulty services or resources.

```elixir
defmodule PaymentService do
  use Brama.Decorator

  @decorate circuit_breaker(identifier: "payment_api")
  def process_payment(payment) do
    PaymentAPI.process(payment)
  end
end
```

### The result

When I first read Geoffrey's article, I was very intrigued. So, I started with technical specifications. I used the approach I use every day and brainstormed it with Claude. I developed a clear set of specs, APIs, and examples. I prompted it appropriately, and in 15 minutes, I got my specs.

The rules part took me a couple of hours. It was a careful process that initially involved some time because I needed to dump my approach to engineering things. I used quite a simplified set of rules. I took some from archived Erlang best practices and compiled them into Cursor rules + my own ones. During the implementation phase, I added some ones, like complete solution and ban using macros. I still want to improve and extract them into a repo to consistently enhance and reuse them in other projects.

**[Example of a rule](https://github.com/ihorkatkov/brama/blob/main/.cursor/rules/elixir-process-design.mdc)**

Then, I wrote down a prompt, stressing that the agent must study rules and specs, implement what is missing, and validate the result by compiling and running type specs and tests. I clicked Enter.

**[SPECS.md](https://github.com/ihorkatkov/brama/blob/main/SPECS.md)**

**Prompt:**
```markdown
Study @SPECS.md for functional specifications.
Study @rules  for technical requirements
Implement what is not implemented
Create tests
Run "mix compile --warnings-as-errors" and verify the application works
Run "mix credo" and resolve linting errors
Run "mix test" and resolve test failures
Run "mix dialyzer" and resolve dialyzer warnings

```

It was very exciting to see how it delivered the result. It implemented the first version, including tests, in a few minutes. It tried to compile, but obviously, it didn't work. Then, it considered for a moment what went wrong and tried to fix it again, and again, and again. Compilation went through, and it tried to run 76 written tests. All of them failed, but slowly, one by one, it fixed the code. And boom, I have a fully functional library.

### Learnings

Despite it being very fun to see it's working, I needed to occasionally click on "continue" because it automatically stops after 25 actions. After trying to fix types or tests three times in a row, it ignored them. Obviously, that was one of the rules I added. Never skip tests, and ensure the result is complete according to specs. The first decorator approach was utterly broken, and all tests were ignored. I didn't mention in the specs that I want to use the decorator library instead of writing my own implementation.That's how another rule appeared -> never write macroses unless implicitly specified. Macroses are hard in Elixir, and of course, it means there are not many code examples on the web where LLMs could learn how to do it properly. I redid the specs with Claude again and asked to reimplement the solution, which worked out.
Now, when I reflect on it, I realize I had to split the work into manageable tasks. For example, I could start with the main functionality first, then the notifications system, and only then the decorators. It would be much easier for me to supervise it, and it would be much easier and faster for the agent to deliver it.

### Revised approach

In the coming weeks, I want to hone the approach. The framework must include a set of prompts for implementation, specs, and rules. It must also include a unified way of validation and probably include fine-tuned LLM, which should validate the agent's end result. The agent engineer is going to talk to the agent reviewer. This is crazy if you consider how fast and far we have come in the last two years.

The execution itself must be automated. Consider integration with a ticketing system like GitHub, where the agent takes tasks and tries to implement them.

Although the agent implemented it in only an hour, it could be improved even further given that there were actually three tasks instead of one. I can easily imagine parallel execution in projects like Cursor in the near future.

### Final thoughts

That experiment made me a believer in agent-augmented development. Our industry is going to change drastically in two years or even faster. Exponentially faster execution will demand businesses to adapt and generate new ideas. Our profession and the way we work are going to look different. We need to find ways to integrate into existing teams and projects. Teams and processes have to adapt.

This paradigm shift will demand new skills from us: becoming better at articulating specifications, codifying engineering principles, and breaking complex tasks into manageable chunks. The engineers who master these meta-skills will lead this transformation.

Should we be afraid of it? Of course not! This isn't the end of software engineering—it's a renaissance.