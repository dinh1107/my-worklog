---
title: "Event 1 - AWS VIETNAM COMMUNITY MEETUP"
date: 2026-07-25
weight: 1
chapter: false
pre: "<b>4.1. </b>"
---

# AWS VIETNAM COMMUNITY MEETUP

## Event Information

| Item | Details |
| --- | --- |
| **Event Name** | AWS VIETNAM COMMUNITY MEETUP |
| **Date** | Saturday, July 25, 2026 |
| **Time** | 08:30 – 12:00 (GMT+7) |
| **Location** | Grand Terra Tower, 36 Cat Linh Street, O Cho Dua, Hanoi |
| **Format** | In person |
| **Role** | Attendee |

---

# Summary Report

## Event Objectives

AWS Vietnam Community Meetup brought together people interested in AWS, artificial intelligence, and the practical application of AI at work. The event offered an opportunity to learn about emerging trends, hear from professionals implementing real-world solutions, and connect with others pursuing cloud and AI careers.

The main objectives were to:

- Introduce the activities and development direction of the AWS community in Vietnam.
- Share how AI agents are built, operated, and applied in practice.
- Explain how AI creates value in software development, infrastructure, and business.
- Discuss how to select an appropriate AI agent pattern on AWS.
- Expand connections among students, engineers, AWS Community Builders, and industry professionals.

---

## Event Agenda

| Time | Session | Speaker(s) |
| --- | --- | --- |
| **08:30 – 09:00** | Check-in and networking | Organizers and attendees |
| **09:00 – 09:20** | Community Introduction | Anh Ho and Phong Pham |
| **09:20 – 09:50** | OpenClaw – The Rise and Practice of Open-Source AI Agents | Tuan Vu |
| **09:50 – 10:20** | From AI Trends to Business Value | Nguyen Thu and Nam La |
| **10:20 – 10:40** | Tea break, group photo, and networking | All attendees |
| **10:40 – 11:10** | Ship Fast with AI, Not by AI | Henry (Duc) Bui |
| **11:10 – 11:40** | Selecting the Right AI Agent Pattern on AWS | Dzung Luong |
| **11:40 – 12:00** | Kahoot Quiz and Lucky Draw | Organizers and attendees |

---

## Speakers

- Anh Ho
- Phong Pham
- Tuan Vu
- Nguyen Thu
- Nam La
- Henry (Duc) Bui
- Dzung Luong

---

# Key Highlights

## 1. Community Introduction

The opening session introduced the growth, knowledge-sharing activities, and member engagement goals of the AWS community in Vietnam. It helped attendees understand how the community supports learning, practical knowledge exchange, and access to AWS programs.

Key points:

- Important milestones in the community's development.
- AWS meetups and technical knowledge-sharing activities.
- Opportunities to connect learners, engineers, and industry professionals.
- Collaboration and knowledge exchange with the international AWS community.

## 2. OpenClaw – The Rise and Practice of Open-Source AI Agents

This session presented OpenClaw as an open-source agent runtime and examined how an AI agent organizes its reasoning loop, uses tools, maintains context, and coordinates multiple tasks.

Key points:

- The concept of an agent runtime and how it differs from a conventional AI application.
- OpenClaw's layered architecture, from model calls to the gateway layer.
- The agent loop: receiving a request, reasoning, calling tools, and responding.
- Expansion into advanced multi-agent use cases.
- Risks involving tool permissions, prompt injection, and skill injection.
- The limitations of “memory” when retained information is mainly added back into the prompt.

## 3. From AI Trends to Business Value

This session focused on the gap between a technology trend and measurable business value. Using sales as an example, the speakers showed that AI is most useful for repetitive preparation, information synthesis, and content assistance.

Key points:

- The differences among AI, Generative AI, and AI agents.
- AI assistance across prospecting, research, discovery, proposal, and follow-up.
- Reduced research and proposal preparation time.
- More time for customer engagement instead of repetitive manual work.
- Trust, relationships, communication, and negotiation remain human responsibilities.

## 4. Ship Fast with AI, Not by AI

This session emphasized that AI can accelerate the write, run, and debug loop, but product delivery still depends on review, testing, integration, deployment, and operations. Development teams therefore need strong processes and supporting infrastructure instead of relying entirely on AI-generated code.

Key points:

- The distinction between the **Inner Loop** and **Outer Loop** of software development.
- Converting recurring knowledge into lint rules, tests, and CI checks.
- Keeping documentation in the repository for both people and agents.
- Building a shared language through living documentation.
- Increasing verification for critical code.
- Keeping humans as the authors who remain accountable for the product.

## 5. Selecting the Right AI Agent Pattern on AWS

Dzung Luong's session focused on choosing an AI agent pattern that fits the problem rather than applying a single architecture to every use case. Designing an AWS-based solution requires balancing task complexity, coordination, control, cost, latency, security, and system observability.

Key points:

- Start with business requirements and the autonomy the agent genuinely needs.
- Distinguish use cases for a single agent, a sequential workflow, and a multi-agent system.
- Separate responsibilities when a problem needs multiple specialized capabilities.
- Prefer controlled workflows for tasks requiring stable and predictable execution.
- Observe prompts, tool calls, execution state, latency, and cost.
- Apply least privilege to the data and tools available to an agent.
- Keep human review or approval for high-risk actions.

---

# Key Takeaways

## Select the agent pattern based on the problem

Not every task requires a multi-agent system. A straightforward process with clear inputs and outputs is often better served by a predefined workflow or a single agent. Multi-agent architectures add value when the problem genuinely requires specialized roles, complex coordination, or parallel execution.

## Faster coding does not automatically mean faster delivery

AI can significantly reduce the time required to write and debug code, while review, testing, integration, and operations may remain bottlenecks. Improving end-to-end productivity therefore requires optimizing the Outer Loop as well as code generation.

## Knowledge should become shared infrastructure

Recurring rules and common failures should be captured in documentation, lint rules, automated tests, or CI checks. This turns individual knowledge into a reusable asset for the entire team and its AI agents.

## Observability and security are mandatory

Agents can call tools and affect real data, so the system must make their execution traceable, restrict access, and control sensitive actions. Explainability, logging, and human approval are essential for safe agent operations.

## AI helps people create more value

AI creates practical value by reducing repetitive work and assisting with research and information synthesis. This gives people more time for judgment, creativity, decision-making, and relationship building. AI does not replace human accountability or trust.

---

# Event Experience

Attending AWS Vietnam Community Meetup was a meaningful experience in my AWS, cloud, and AI learning journey. The event did more than introduce new trends; it showed how the same technologies are evaluated from software development, infrastructure, business, and risk-management perspectives.

I was particularly impressed by the message of “Ship Fast with AI, Not by AI.” AI can help people write code faster, but a reliable product still requires review, testing, documentation, and human accountability. The session on selecting the right AI agent pattern on AWS also helped me understand that the best architecture is not the most complicated one, but the one that best fits the requirements, budget, and required level of control.

The event also allowed me to meet people who are studying and working in cloud and AI. These conversations gave me practical perspectives, helped me identify skills I should continue developing, and clarified the direction of my future learning.

---

# Lessons Learned

- Define the business problem before choosing a technology or agent pattern.
- Use a multi-agent design only when its specialization and coordination benefits justify the added complexity.
- Treat AI-generated code as input that still requires review, testing, and monitoring.
- Capture team knowledge in the repository, living documentation, and automated checks.
- Apply least-privilege access and maintain complete execution logs.
- Preserve human approval for important decisions and potentially high-risk actions.
- Combine cloud, AI, security, and communication skills to create practical value.

---

# Event Photos



![AWS VIETNAM COMMUNITY MEETUP](/images/4.1.jpg)



---

# References

- [Event information and agenda](https://luma.com/kmx59nxa)
- [Session videos and materials](https://drive.google.com/drive/folders/1P5TGC-_-Z_t4Z7Di9m5DGLy2gugElywE)

