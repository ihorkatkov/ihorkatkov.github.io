---
layout: post
title: "BEAM and Team: Your Playbook for Hiring Elixir Developers"
date: 2024-08-29
categories: [Elixir, Hiring]
tags: [elixir, hiring, software-engineering]
author: Ihor Katkov & Sofiia Yurkevska
---

In my various roles, I've hired and worked with many Elixir developers. I've also seen many companies struggle with hiring Elixir developers. In this post, I'll share my experience and insights into hiring engineers in general and Elixir developers in particular. The article was written in collaboration with [Freshcode](https://freshcode.com)

# BEAM and Team: Your Playbook for Hiring Elixir Developers

Finding the right talent can make or break your company’s digital initiatives. This guide offers strategic insights into hiring the best developers for those considering **Elixir** for their next project.

---

## Map Out Your Business and Technology Strategies

Before diving into the hiring process, aligning your technical strategy with your overall business goals is crucial. Elixir, known for its scalability and fault tolerance, is particularly well-suited for:

- Building highly concurrent and distributed systems  
- Developing real-time applications with low latency  
- Handling large volumes of data efficiently  
- Implementing real-time systems  

To illustrate how these capabilities translate to real-world scenarios, consider this brief case study.

### A Telecom Company’s Journey

A telecom company providing Internet and IoT solutions struggled with its Java infrastructure. Their system managed 50,000 connected devices but could only handle 5,000 per server, creating a scalability bottleneck. This bottleneck was compounded by:

- Slow SLA management
- Limited IoT integration capabilities
- High development costs
- Protracted three-month release cycles for updates and new features

After analysis, they transitioned to Elixir. This choice aligned well with their specific needs:

- **Elixir’s actor model and OTP framework** were ideal for managing numerous concurrent connections and addressing device scaling issues.
- The **OTP framework’s supervisor trees** helped improve system resilience, crucial for maintaining SLAs.
- Elixir’s functional nature facilitated easier testing and maintenance, reducing development costs.
- **Hot code reloading** enabled updates with minimal downtime, significantly shortening release cycles.
- Tools like **Nerves for IoT** and **Broadway for data ingestion** streamlined custom integrations.

The transition team consisted of two Elixir developers, a project manager, a business analyst, and a QA engineer. This focused team structure allowed for efficient knowledge transfer and system development.

#### What Can We Learn?

- **Start with your current challenges:** Identify your technical pain points before making decisions for technology or hiring.
- **Think long-term:** Consider how a technology aligns with your future scalability and maintenance needs.
- **Consider team composition:** The right team isn’t always about hiring many specialists but about having the right mix of skills.
- **Plan for knowledge transfer:** Consider how you’ll build and maintain expertise in new technologies within your organization.
- **Be prepared for challenges:** Any tech transition will have hurdles. Plan for them in advance.
- **Choose between local and remote:** Evaluate whether remote hiring can expand your talent pool or if investing in local talent development better suits your needs.

It’s worth noting that while Elixir offers significant benefits, the talent pool is smaller compared to mainstream languages. You may find fewer readily available experienced developers, potentially higher costs for top talent, and a longer recruitment process. However, this scarcity also comes with significant upsides. Elixir’s community is known for its **high skill level and passion for the language**, which can translate into innovation and cutting-edge solutions for your company.

---

## Create Your Ideal Candidate Profile

When building your Elixir team, look beyond just technical skills. The ideal candidate brings a combination of technical prowess, problem-solving abilities, and the right mindset. Here’s what to prioritize in your search:

1. **Engineering and Growth Mindset**  
   Look for problem-solving skills, analytical thinking, and a solid technical foundation. These traits often indicate the ability to tackle complex challenges. A growth mindset suggests a willingness to learn and adapt.

2. **Functional Programming Experience**  
   While direct Elixir experience is valuable, don’t overlook candidates with backgrounds in other functional programming languages (e.g., Haskell, Clojure, Lisp). If they can explain monads, hire them. If they can explain monads in a way that makes sense, they’re probably wizards!

3. **Specific Technical Skills**  
   - Experience with OTP for building distributed systems.  
   - Familiarity with Phoenix and LiveView (part of the PETAL stack) for web projects.  
   - Tailor technical requirements to your specific goals.

4. **GitHub Presence**  
   A candidate’s GitHub profile can reveal coding style, quality of work, and passion for programming. Look for active contributions to open-source projects, especially those related to Elixir or functional programming.

Remember: **mindset often trumps specific language experience**. Be prepared to invest in training promising candidates with strong fundamentals.

---

## Source Elixir Expertise

Finding top-tier Elixir developers requires a multi-faceted approach. Given the relatively niche nature of Elixir, traditional hiring methods may fall short. Here’s a roadmap:

1. **Start Where Elixir Developers Congregate**  
   - [ElixirForum](https://elixirforum.com)  
   - [ElixirJobs](https://elixirjobs.net)  
   - LinkedIn, with targeted searches and networking strategies.

2. **Tap Into Personal Networks**  
   Encourage your current engineering team to reach out to their networks. An employee referral program can sweeten the deal.

3. **Build a Community Presence**  
   - Host or sponsor Elixir conferences and meetups.  
   - Contribute to Elixir open-source projects.  
   - Engage with the Elixir community to become a recognized supporter.

4. **Consider Agencies—With Caution**  
   Specialized agencies can be helpful, but be wary of resume flooding. Ensure any agency understands your requirements, culture, and Elixir’s nuances.

---

## Assess the Candidates

The first Elixir hire is often the most challenging and sets the foundation for your Elixir initiatives. Ideally, you want someone with deep technical expertise, mentoring capabilities, and strong architectural insight.

### Technical Evaluations

- **Real-World Coding Challenges**: Instead of abstract problems, create tasks that mirror your actual projects.
  - Implement a simple pub-sub system in Elixir (for real-time apps).
  - Build an ETL workflow orchestrator with scheduling, logging, and fault tolerance using OTP principles.
  - Develop a basic Phoenix application with specified features.

- **Pair Programming Sessions**: Observe problem-solving approaches, communication skills, and teamwork potential.

- **System Design Challenge (for Senior Roles)**: Present a high-level business problem and ask candidates to design a scalable solution with Elixir and OTP.

### The “Second 80%”

After confirming they can code, check if they can align with your team and stakeholders:

- **Evaluate Commitment to Growth**  
  - Past contributions to open-source Elixir projects.  
  - Recent Elixir features or libraries they’ve learned.  
  - Methods for staying updated in the Elixir ecosystem.

- **Trial Day**  
  - Offer a day (paid) for the candidate to work on small, real tasks with your team.  
  - Helps gauge cultural fit and teamwork style.

**Freshcode Tip**: Leverage your in-house expertise if you have Elixir veterans:

- Have them review the candidate’s code.  
- Include them in technical interviews.  
- Assess the candidate’s grasp of OTP, concurrency models, and essential Elixir libraries.

---

## Common Pitfalls to Avoid

- Overemphasizing Elixir-specific experience at the expense of general programming skills.  
- Neglecting soft skills and cultural fit.  
- Conducting inadequate technical evaluations.  
- Overlooking remote talent in a distributed world.  
- Skimping on onboarding and mentoring.  
- Forgetting about community engagement (Elixir enthusiasts often come in packs!).

---

## Onboard ’Em Well

Once you’ve successfully hired Elixir developers, a solid onboarding process is key. Here’s how to do it right:

1. **Facilitate Quick Code Contributions**  
   - Have new hires make their first commit within the first week.  
   - Boosts confidence and motivation, and gives you immediate feedback on their performance.

2. **Implement a “Test Day” Approach**  
   - A paid day where the candidate tackles small, real tasks in your environment.  
   - Lets both parties gauge fit and collaboration style.

3. **Establish a Robust Mentorship Program**  
   - Assign each new Elixir developer a dedicated mentor.  
   - Guide them through your coding practices, workflows, and codebase intricacies.  
   - Schedule regular check-ins and milestones.

4. **Create a Comprehensive Onboarding Plan**  
   - Roadmap of tasks and projects over the first few months.  
   - Scheduled intros to teams and stakeholders.  
   - Training on your specific tech stack and tools.  
   - Regular feedback sessions to address challenges or questions.

---

## Final Words

Hiring Elixir developers requires a strategic approach that balances technical skills, cultural fit, and business objectives. By following these guidelines and considering all available options, including strategic outsourcing partnerships, you can build a high-performing Elixir team that drives your business forward.

Co-authored with [Sofiia Yurkevska](https://www.linkedin.com/in/s-yurkevska/) |
Originally posted in collaboration with [Freshcode](https://www.freshcodeit.com/blog/strategic-software-migration-reasons-risks-best-practices)