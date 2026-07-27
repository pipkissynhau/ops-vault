---
title: Day2 - Phase 1 - The Differences Between LLMs and Traditional Programs
tags: [Large Models, LLMs, AI Basics, Introduction to Large Models, Traditional Programs, Ops Engineers]
date: 2026-03-28
---

# Day2 - Phase 1 - The Differences Between LLMs and Traditional Programs

## I. Today's Learning Objectives

Today, this article will focus on addressing a very critical question:

**What are the fundamental differences between large language models (LLMs) and traditional programs?**

After reading this, you should be able to do the following:

1. Understand what an LLM is.
2. Comprehend the core differences in how LLMs and traditional programs function.
3. Identify scenarios where scripts/programs are suitable and when introducing LLMs makes sense.
4. Establish a clear distinction between "deterministic tasks" and "semantic tasks" from an operations perspective.
5. Avoid treating LLMs as rule engines or scripts as universal intelligent agents.

We will not delve into complex training principles today.  
The focus of this article is:

**To first clearly distinguish between 'programs' and 'models'**

---

## II. What is an LLM

LLM stands for Large Language Model, which in Chinese is commonly referred to as:

**Large Language Model.**

You can initially understand it as:

**A large model specifically designed for natural language understanding and generation.**

There are two key terms here:

### 1. Language
It primarily deals with language, including:

- Text
- Questions
- Conversations
- Documents
- Log descriptions
- Descriptive content

### 2. Model
It is not a set of fixed if/else rules but a trained model system.

Therefore, an LLM is not an "advanced script."  
Essentially, it is a:

**Model that performs language understanding and generation based on its training results.**

Many AI applications such as dialogue systems, document summaries, code generation, and log analysis are likely powered by LLMs behind the scenes.

---

## III. Why Do Many People Confuse LLMs with Programs

On the surface, both can "process input and produce output."

For example:

### Traditional Programs
Input a log, and it outputs whether it contains an ERROR.

### LLMs
Input a log, and it outputs a summary of the issue, possible causes, and troubleshooting suggestions.

Both seem to follow the pattern of "input -> processing -> output."

However, their underlying processing logic is entirely different.

This is the most important point to understand today.

---

## IV. How Traditional Programs Work

The core of traditional programs is:

**Humans define the rules first, and the program strictly follows them.**

For instance:

    if "ERROR" in log_line:
        print("Error log")

Or:

    if cpu_usage > 90:
        print("Alert triggered")

These programs have clear characteristics:

- Rules are explicitly written by humans.
- The same input usually results in the same output.
- The outcomes are stable and reproducible.
- They are easy to test and audit.
- They are suitable for automated execution.

You can think of traditional programs as:

**Deterministic execution systems.**

As long as the rules are clear, they will perform reliably.

---

## V. How LLMs Work

The core of an LLM is not "writing every rule manually" but:

**Learning vast language patterns during training and then generating the most likely reasonable output based on the current context.**

For example, if you give it a log saying:

"Please analyze this log and determine the possible cause of the issue."

It won’t just match a few keywords like a script would.  
Instead, it might consider:

- What errors are mentioned in the log.
- Whether these errors are related to each other.
- Which statement is more likely the root cause and which is a secondary effect.
- What such errors typically indicate in common systems.
- Whether to check network, permissions, configuration, or dependent services first.

Therefore, LLMs are more like:

**Systems that perform semantic understanding and content generation based on context.**

This is why they can handle many tasks that traditional scripts struggle with.

---

## VI. A One-Sentence Distinction Between the Two

You can start by remembering this comparison:

### Traditional Programs
**Rule-driven.**

### LLMs
**Pattern learning + probabilistic generation-driven.**

To put it more plainly:

### Traditional Programs
"I will do exactly what you tell me to."

### LLMs
"Give me the context, and I will provide a likely and reasonable answer based on my trained patterns."

These two are not competing with each other but rather:

**Handle different types of problems.**

---

## VII. From an Operations Perspective, What Are Deterministic Tasks

"Deterministic tasks" can be understood as:

**Tasks where the rules are clear, the judgment criteria are definiteMany times, it is still necessary to rely on retrieval systems, knowledge bases, databases, and monitoring systems to provide accurate information, which are then understood and organized by LLMs.

---

## Fifteen, The Key Takeaways of Today’s Article

Today, you should focus on remembering the following points.

### 1.
**An LLM is a large language model whose core ability lies in processing natural language and generating content.**

### 2.
**Traditional programs operate based on rules, while LLMs learn from trained patterns and generate outputs probabilistically.**

### 3.
**Deterministic tasks are better suited for scripts and rule-based systems, whereas semantic tasks are more appropriate for LLMs.**

### 4.
**For high-risk actions, one cannot rely solely on LLMs; rules, permissions, and confirmation mechanisms are essential.**

### 5.
**In engineering, the correct approach is usually not a binary choice but rather “programs handle deterministic tasks, while LLMs handle semantic aspects.”**

---

## Sixteen, Today’s Self-Testing Questions

Before going to bed, you can review these questions.

1. What is an LLM?
2. What is the main difference in how LLMs and traditional programs work?
3. What are deterministic tasks?
4. What are semantic tasks?
5. Why cannot high-risk actions be entrusted solely to LLMs?
6. In log analysis, what are the suitable applications for scripts and LLMs?

If you can explain these questions in your own words, then you have understood today’s content.

---

## Seventeen, Tomorrow’s Preview

Tomorrow we will continue with:

**Day3 - Phase One: Tokens, Context, and Context Windows**

The key topics tomorrow will include:

- What exactly is a token?
- Why don’t models understand content based on “entire sentences”?
- What is context?
- What is a context window?
- Why do models have difficulty remembering longer conversations?
- Why is the processing of long documents limited by context windows?

---

## Eighteen, Additional Reading

You can familiarize yourself with the following materials for now, but there is no need to read them in depth immediately.

- Hugging Face LLM Course  
  https://huggingface.co/learn/llm-course

- Text and dialogue-related explanations in OpenAI’s API documentation  
  https://platform.openai.com/docs

- LangChain’s introduction to concepts such as LLMs, Prompts, Memory, and Agents  
  https://python.langchain.com/docs/introduction/

---

## Nineteen, A Final Thought

**Scripts are used for problems where “rules are clear,” while LLMs are used for problems that require “language understanding.”**  
Establishing this distinction will help you avoid confusion when learning about Prompts, RAG, and Agents in the future.