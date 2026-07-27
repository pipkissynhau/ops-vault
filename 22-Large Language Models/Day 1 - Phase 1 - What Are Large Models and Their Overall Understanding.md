---
title: Day1-Phase 1-What are Large Models and Their Overall Understanding
tags: [Large Models, LLMs, AI Basics, Introduction to Large Models, AI Platforms, Operations Engineers]
date: 2026-03-28
---

# Day1-Phase 1-What are Large Models and Their Overall Understanding

## I. Today's Learning Objectives

Today, we will focus on one thing:

**First, establish an overall understanding of what “large models” are.**

After reading this, you should be able to do the following:

1. Understand what large models are.
2. Recognize the main differences between large models and traditional programs.
3. Know what large models can and cannot do.
4. Get a basic idea of concepts like Tokens, Prompts, RAG, and Agents that we will learn later.

Today, we won't delve into complex theories, training formulas, or code implementation.  
First, get a clear understanding of the basics before moving on.

---

## II. What are Large Models

The term “large model” usually refers to:

**Models with a very large number of parameters that are trained using massive amounts of data and can process language or multiple modalities of information.**

In today's context, the most common types include:

- Large Language Models (LLMs)
- Multimodal Models (capable of handling text, images, audio, etc.)

For now, you can think of a large model as follows:

**It is a highly capable “language understanding and generation system.”**

Its typical capabilities include:

- Conducting conversations
- Summarizing content
- Revising texts
- Translating languages
- Writing code
- Analyzing text
- Answering questions based on context

Therefore, large models are not just “chatbots.”  
Chatting is one of their applications; fundamentally, they are systems designed to handle language tasks.

---

## III. How Do Large Models Differ from Traditional Programs

This is one of the most important concepts to understand today.

### 1. Traditional Programs

The core of traditional programs is:

**Humans write rules first, and the program follows these rules.**

For example, an operations script might look like this:

    if CPU > 90:
        Trigger an alarm

Or a log analysis script:

    if "ERROR" in log_line:
        Mark it as an error log

These programs have the following characteristics:

- Clear rules
- Predictable results
- Repeatability
- Suitable for deterministic tasks

In other words, **traditional programs excel at tasks with clear rules.**

---

### 2. Large Models

Large models are different.

They don't rely on pre-written rules but learn “patterns” and “probabilistic relationships” from large amounts of data.  

So, they are more like:

**You give them a task, and they generate the most likely reasonable answer based on context and what they have learned.**

For example, if you provide them with a log, they can:

- Summarize the main issues
- Propose possible causes
- Offer troubleshooting steps
- Rewrite informal descriptions into formal fault reports

Therefore, their strengths lie in:

**Semantic understanding, ambiguous judgment, content generation, and text organization.**

---

### 3. A Quick Comparison

You can remember it this way:

**Traditional programs are rule-driven.**  
**Large models generate answers based on data training.**

This also explains why large models can sometimes be very intelligent but also make mistakes.

---

## IV. What Exactly Do Large Models Do?

Without getting too technical, you can think of it this way:

**The core ability of a large model is to predict the most reasonable content based on context.**

For example, if you enter a sentence like:

“Please help me analyze this Kubernetes log and provide troubleshooting suggestions,”

the model will consider:

- The sentence itself
- The attached log
- The knowledge it has learned from training

and then generate subsequent content step by step.

So, although it seems to “think,” at a fundamental level, it is performing a highly complex process of **context-based content prediction.**

You don’t need to fully grasp this concept right away; just remember that:

**Large models are not magic—they are still programs, but their capabilities come from trained parameters, not hard-coded rules.**

---

## V. What Can Large Models Do?

From a practical perspective for your future studies and work, large models have the following common capabilities:

### 1. Text Understanding
- Understand user questions
- Analyze log descriptions
- Comprehend document content
- Interpret fault reports

### 2. Text Generation
- Create summaries
- Produce reports
- Write scripts
- Generate troubleshooting steps
- Format documents in Markdown

### 3. Text Revision
- Convert informal language into formal expressions
- Organize scattered information into structured documents
- Optimize resume content for professional use

- Andrej Karpathy - Neural Networks: Zero to Hero  
  https://karpathy.ai/zero-to-hero.html

---

## Fifteen, Concluding with One Sentence

**Think of large models as a new kind of “foundation for language capabilities,” rather than some mysterious magic.**  
This way, learning about Tokens, Prompts, RAG, and Agents will become much smoother as you progress.