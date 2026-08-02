---
title: "# Day2-First Stage-LLM vs Traditional Programs"
tags: "[Large Model, LLM, AI Fundamentals, Introduction to Large Models, Traditional Programs, Operations Engineer]"
date: 2026-03-28
---

# Day2 - Phase 1 - Differences Between LLM and Traditional Programs

## I. Today's Learning Objectives

This article primarily addresses a very critical question:

**What is the fundamental difference between large language models (LLM) and traditional programs?**

After reading, you should be able to:

1. Understand what an LLM is
2. Grasp the core differences in how LLM and traditional programs operate
3. Know which scenarios are suitable for scripts/programs and which are suitable for LLM
4. Establish the boundary between "deterministic tasks" and "semantic tasks" from an operations perspective
5. Avoid treating LLM as a rule engine or scripts as universal intelligences

This article still does not cover complex training principles.  
The focus of this article is:

**First, clearly distinguish between "programs" and "models" as two different categories.**

---

## II. What is an LLM

LLM is an abbreviation for Large Language Model, commonly referred to as:

**Large Language Model.**

You can initially understand it as:

**A large model specialized in natural language understanding and generation.**

The key terms here are two:

### 1. Language
It primarily processes language, which includes:

- Text
- Questions
- Conversations
- Documents
- Log explanations
- Descriptive content

### 2. Model
It is not a set of fixed if/else rules, but rather a trained model system.

Therefore, LLM is not "advanced scripting."

At its core, it is:

**A model that generates language understanding and creation based on training results.**

Many of the AI dialogues, document summaries, code generation, and log analysis you see today essentially rely on LLMs working behind the scenes.

---

## III. Why Many People Confuse LLM with Programs

Because at first glance, they both can "process input and produce output."

For example:

### Traditional Program
Input a log line, output whether it contains "ERROR."

### LLM
Input a log line, output an exception summary, possible causes, and troubleshooting suggestions.

They both appear to be "input -> processing -> output."

But their underlying processing logic is fundamentally different.

This is the most important understanding point for today.

---

## IV. How Traditional Programs Work

The core of traditional programs is:

**Humans first define rules, and the program strictly follows these rules.**

For example:

    if "ERROR" in log_line:
        print("Error log")

Another example:

    if cpu_usage > 90:
        print("Trigger alert")

The characteristics of such programs are very clear:

- Rules are explicitly written by humans
- Same input usually produces the same output
- Results are stable and reproducible
- Easy to test
- Easy to audit
- Suitable for automation execution

You can understand traditional programs as:

**Deterministic execution systems.**

As long as the rules are clear, they will be very reliable.

---

## V. How LLM Works

The core of LLM is not "you write every rule," but rather:

**It learns a large amount of language patterns during training, then generates the most probable reasonable output based on the current input context.**

For example, you give it a log:

"Please help me analyze the following log and determine the possible source of the problem."

It won't match specific keywords like a script does.  
It may comprehensively judge:

- What errors appear in the log
- Whether the errors are related
- Which line is more likely the root cause and which is a chain reaction
- What these errors typically mean in common systems
- Whether to prioritize checking network, permissions, configuration, or dependent services

Therefore, LLM is more like:

**A system that performs semantic understanding and content generation based on context.**

This is why it can do many things that traditional scripts are not good at.

---

## VI. A One-Sentence Distinction Between the Two

You can first remember this comparison:

### Traditional Program
**Rule-driven.**

### LLM
**Pattern learning + probability generation-driven.**

Even more straightforwardly:

### Traditional Program
"You write the rules, and I follow them exactly."

### LLM
"You give me context, and I generate a probable reasonable answer based on patterns I learned during training."

These two are not replacements for each other, but rather:

**They handle different types of problems.**

---

## VII. What is a Deterministic Task from an Operations Perspective

"Deterministic tasks" can initially be understood as:

**Tasks with clear rules, explicit judgment criteria, and the need for stable and consistent results.**

Such tasks are very suitable for scripts, programs, and rule engines.

Examples of such scenarios include:

### 1. Status Judgment
- Whether a Pod is Running
- Whether a service port is listening
- Whether a process exists
- Whether a file exceeds 10G

### 2. Fixed Rule Classification
- HTTP status code classification (2xx / 4xx / 5xx)
- Log classification based on ERROR / WARNING / INFO
- Disk usage exceeding a threshold triggers an alert
- Certificate expiration time less than 30 days triggers a notification

### 3. Fixed Process Execution
- Log archiving
- File backup
- Service restart
- Patrol result summary
- Scheduled cleanup of temporary files

### 4. High-Risk Control
- Permission validation
- Deletion action confirmation
- Change approval process
- Production environment release access control

The commonality of these tasks is:

**They cannot be ambiguous, and they cannot rely on "guessing."**

---

## VIII. What is a Semantic Task from an Operations Perspective

"Semantic tasks" can initially be understood as:

**Tasks that require understanding text, context, intent, and meaning rather than just matching rules.**

Such tasks are more suitable for LLM.

Examples include:

### 1. Log Summarization
Give the model dozens of error log lines, let it summarize the main issue.

### 2. Fault Description Organization
Organize a conversational troubleshooting process into a formal fault report.

### 3. Runbook-Assisted Q&A
Based on document content, answer:
"What should be checked first when Redis master-slave delay is high?"

### 4. Work Order Classification
Based on user-submitted problem descriptions, determine whether it's more likely a network, middleware, or application issue.

### 5. Alert Interpretation
Translate technical alerts like "etcd request timeout" into more understandable explanations.

### 6. Document Rewriting
Organize scattered notes into structured SOPs, weekly reports, or post-mortem documents.

The commonality of these tasks is:

**They require understanding content, not just matching rules.**

---

## IX. Differences Between Programs and LLM in the Same Scenario

Use an example you can easily relate to.

### Scenario: Analyze a Batch of Logs

#### What Traditional Programs Can Do
For example:

- Count the number of ERRORs
- Filter logs containing "timeout"
- Extract 5xx status codes
- Count occurrences of each error
- Classify by keywords

These are very suitable for scripts.

Because the rules are clear, the results are stable, and the speed is fast.

#### What LLM Can Do
For example:

- Summarize the main fault phenomena reflected in these logs
- Infer the most likely root cause
- Distinguish between "core errors" and "secondary errors"
- Generate troubleshooting suggestions
- Output suitable reporting language for leaders or teams

These are more aligned with LLM's strengths.

#### Best Practice
Usually, it's not an either-or choice, but rather:

**First use programs for cleaning, filtering, extraction, and aggregation, then pass the results to LLM for understanding, summarization, and rewriting.**

This is a more reasonable engineering approach.

---

## X. Why LLM Cannot Directly Serve as a Rule Engine

This is a particularly common pitfall.

You'll notice that LLM can sometimes "judge," so many people think:

"Can I just let it decide everything?"

This is dangerous.

There are several reasons.

### 1. Its output is not necessarily stable
The same sentence, phrased differently, may produce different results.

### 2. It may misinterpret boundary conditions
For example, a log line containing "error" may actually be historical playback and not indicate a current fault.

### 3. It Might Fabricate  
Especially when information is insufficient, it may generate an "appearing reasonable" answer.

### 4. It Is Not Naturally Auditable  
Unlike fixed rules, it is not easy to trace "why this judgment was made."

Therefore, when dealing with the following matters, you cannot rely solely on LLM:

- Whether to delete data  
- Whether to execute commands directly  
- Whether to determine permission approval  
- Whether to allow production changes  
- Whether to automatically close alerts  
- Whether to automatically restart critical services  

These areas are better suited for:

**Rule systems to make the final judgment, with LLM providing auxiliary suggestions only.**

---

## Eleven. Why Traditional Programs Cannot Replace LLM  

The reverse is also true.  

Some people think:  
"Once you write enough scripts, you can do anything, right?"  

This is not realistic.  

Because many things cannot be covered by simple rules.  

For example:  

### 1. User Query Styles Are Not Fixed  
The same meaning may be expressed in many different ways.  

### 2. Fault Descriptions Are Very Conversational  
Users do not speak strictly by keywords.  

### 3. Document Content Is Scattered  
Much information is hidden in different documents and paragraphs, not easily solved by simple grep.  

### 4. Contextual Understanding Is Required  
The same timeout can be caused by network, DNS, permissions, dependent services, or resource bottlenecks.  

### 5. Natural Expression Is Needed  
Writing summaries, reviews, weekly reports, or emails is not something rule matching can easily accomplish.  

Therefore:  

**Scripts are strong in determinism; LLMs are strong in semantic understanding.**

---

## Twelve. A More Practical Judgment Standard  

When you encounter a scenario and are unsure whether to use scripts or LLM, you can first ask yourself these questions.  

### 1. Is the Judgment Standard Clear?  
If it is very clear, prioritize scripts.  

### 2. Is the Output Required to Be Stable and Consistent?  
If it must be stable and consistent, prioritize scripts.  

### 3. Does It Involve High-Risk Actions?  
If it involves high-risk actions, prioritize rule systems and cannot rely solely on LLM.  

### 4. Is Natural Language or Long Text Understanding Required?  
If so, LLM has an advantage.  

### 5. Is Summary, Rewriting, or Suggestions Needed?  
If so, LLM is more suitable.  

### 6. Can It Be Structured First and Then Passed to the Model?  
If yes, the results are usually better.  

Therefore, the common engineering path is often:  

**Scripts handle "data collection, rule execution, and risk control," while LLM handles "understanding, generation, and auxiliary tasks."**

---

## Thirteen. Understanding This From a Career Development Perspective  

From your direction, the most valuable thing is not to"Concerning.":  
"Which is better, programs or models?"  
Instead, you should establish this awareness:  

**Many future systems will become a combination of "program systems + LLM capabilities."**

For example:  

### 1. Patrol Platform  
- Programs collect metrics  
- Rules make threshold judgments  
- LLM handles anomaly explanations and summaries  

### 2. Log Platform  
- Programs perform filtering, aggregation, and indexing  
- LLM conducts semantic analysis and fault summaries  

### 3. Work Order System  
- Programs handle workflow and permission control  
- LLM performs classification, summaries, and reply suggestions  

### 4. Knowledge Base System  
- Programs handle storage, retrieval, and permissions  
- LLM handles Q&A and content organization  

Your value will increasingly be reflected in:  

**Understanding traditional operations systems and knowing how to reasonably integrate LLM capabilities.**  

This is more practically meaningful than simply "being able to chat."

---

## Fourteen. The Most Common Misconceptions at This Stage  

### Misconception 1: Believing LLM is more advanced than scripts, so everything should be handed over to it  
Wrong.  

Advanced does not mean suitable for all tasks.  

---

### Misconception 2: Believing that anything scripts can do, LLM can definitely do stably  
Not necessarily.  

LLM may be able to do it, but it doesn't mean it's stable, controllable, or auditable.  

---

### Misconception 3: Believing scripts are outdated and that learning AI alone is sufficient  
Wrong.  

The future will actually require:  
- Scripting skills  
- Automation capabilities  
- Interface calling skills  
- Risk control capabilities  
- Systems engineering skills  

Because LLMs need these foundations to be truly implemented.  

---

### Misconception 4: Believing LLM can directly replace search engines or knowledge bases  
This is also inaccurate.  

Often, you still need retrieval systems, knowledge bases, databases, and monitoring systems to provide real information, which LLM then processes and organizes.  

---

## Fifteen. Core Conclusions from Today's Article  

Today, you should remember the following key points.  

### 1.  
**LLM is a large language model, with its core capability being processing natural language and generating content.**  

### 2.  
**Traditional programs rely on rule execution, while LLMs rely on pattern learning and probabilistic generation after training.**  

### 3.  
**Deterministic tasks are better suited for scripts and rule systems, while semantic tasks are better suited for LLMs.**  

### 4.  
**High-risk actions cannot rely solely on LLM; they must have rules, permissions, and confirmation mechanisms.**  

### 5.  
**In engineering, the correct direction is usually not a choice between two options, but "scripts handle determinism, while LLMs handle semantic capabilities."**  

---

## Sixteen. Today's Self-Test Questions  

Before bed, you can go through the following questions.  

1. What is LLM?  
2. What is the biggest difference in working methods between LLM and traditional programs?  
3. What is a deterministic task?  
4. What is a semantic task?  
5. Why can't high-risk actions be solely entrusted to LLM?  
6. In log analysis scenarios, what are scripts and LLMs better suited for?  

If you can explain these questions in your own words, today's article is considered learned.  

---

## Seventeen. Tomorrow's Preview  

Tomorrow will continue with:  
**Day3-First Stage-Token, Context, and Context Window**  

Tomorrow's focus will be on:  
- What exactly is a token  
- Why models don't understand content by "whole sentences"  
- What is context  
- What is a context window  
- Why models "forget" when conversations get longer  
- Why long document processing is limited by the window size  

---

## Eighteen. Supplementary Reading  

Below are resources you can know in advance, without requiring in-depth reading now.  

- Hugging Face LLM Course  
  https://huggingface.co/learn/llm-course  

- OpenAI API Documentation's Text and Conversation Related Notes  
  https://platform.openai.com/docs  

- LangChain's Introduction to LLM, Prompt, Memory, and Agent Concepts  
  https://python.langchain.com/docs/introduction/  

---

## Nineteen. One-Sentence Conclusion  

**Scripts solve the problem of "clear rules," while LLMs solve the problem of "needing to understand language."**  
Establishing this boundary first will prevent confusion when learning Prompt, RAG, and Agent later.