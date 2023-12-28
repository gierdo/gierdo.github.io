---
title: Local AI assistant - codellama vs mistral
categories:
  - programming
  - linux
tags:
  - openai
  - ai
  - mistral
  - llama
  - AMD
---

# Update: Local AI assistant - codellama vs mistral

You might have read my previous posts about [using a local codellama for a vim
coding assistant]({% link _posts/2023-11-24-llama-vim.md %}), setting it up to
use [my laptop's GPU for inference]({% link
_posts/2023-12-16-llama-vim-igpu-amd.md %}) and the [performance evaluation]({%
link _posts/2023-12-17-llama-vim-igpu-amd-performance.md %}).

There are many available language models, so now we have the agony of choice:
Which do we chose? Which is best for the specific usecase?

This article is about my selection process. It is not a complete scientific
comparison of all available language models, and I ran more tests than the ones
documented. However, all results were boringly consistent, and made my decision
pretty straightforward.

There are already [extensive sources and
comparisons](https://www.e2enetworks.com/blog/mistral-7b-vs-llama2-which-performs-better-and-why),
only a quick online search away.

## Environment

The general setup is described already in previous articles, mainly [the
performance evaluation of GPU assisted inference]({% link
_posts/2023-12-16-llama-vim-igpu-amd.md %})

Here is the short summary:

- Lenovo Thinkpad t14s gen 4
- llama.cpp with ROCm on AMD Ryzen 7 PRO 7840U w/ Radeon 780M Graphics, 8G VRAM

This results an boundary conditions for the models to chose from: The models
have to be usable with llama.cpp and cope with 8GB of VRAM.

Which leaves basically only variants of llama2 in the 7b variant, and
mistral-7b.

These two models are compared against each others:

- codellama-7b-instruct.Q4_K_M.gguf
- mistral-7b-instruct-v0.2.Q4_K_M.gguf

## Coding tasks

A part of my workflow and typical use of the AI assistance is coding,
especially the creation of boilerplate, boring stuff which is basically "I know
what to do and how to do it, now I have to write it down.", which takes time
without creating distinct value or inspiration.

## Method

I gave each model a specific task:

```text
Please create a c function for an embedded platform (no stl or time.h, no
POSIX), which returns the unix timestamp for a given time.
```

With the created code in the context, I want to test each model's ability to
work with given code, e.g. for comments and improvement.

```text
What is happening here?

What are potential problems with the given code?

Please give me a well-documented version of this code. Please follow best
practices and do not comment overly verbosely.
```

### codellama

Codellama does not appear to fully understand the request, ignoring the
explicit exclusion of `time.h`.

```text
--------

  Here is an example of how to write a C function that calculates the Unix
  timestamp for a given date:
\`\`\`
#include <stdio.h>

long int get_timestamp(int year, int month, int day) {
    long int timestamp;
    struct tm *tm;

    // Create a new structure to hold the date information
    tm = gmtime(&timestamp);
    tm->tm_year = year - 1900;
    tm->tm_mon = month - 1;
    tm->tm_mday = day;
    timestamp = mktime(tm);

    return timestamp;
}
\`\`\`
This function uses the `gmtime` and `mktime` functions to calculate the Unix
timestamp for a given date. The `gmtime` function takes a pointer to a time
value in seconds since the Epoch (January 1, 1970, 00:00:00 UTC) as an argument
and returns a structure of type `tm`. This structure contains information about
the date and time. The `mktime` function takes a pointer to a `tm` structure as
an argument and returns the number of seconds since the Epoch for the
corresponding date and time.

The `get_timestamp` function first creates a new `tm` structure by calling
`gmtime`. It then sets the year, month, and day values in the structure using
the given parameters. Finally, it calls `mktime` to calculate the Unix
timestamp for the specified date and returns the result.

To use this function, you would call it with the desired date as arguments,
like so:
\`\`\`
long int timestamp = get_timestamp(2018, 3, 24);
printf("Timestamp is %ld\n", timestamp);
\`\`\`
This would print out the Unix timestamp for March 24, 2018.

--------
```

Not ideal, while the code itself is not the real issue.

The same


## General non-coding tasks

I wanted to compare mistral and codellama for general non-coding tasks, which
are just as important for my typical use of AI assistance.

### Method

I put the following bullet points in the context of each model, and asked the
modell to write the given points in full sentences for a short tech article.

- mistral is okay with coding tasks
- mistral appears to be good at general language tasks
- codellama appears to be good at coding tasks, with good knowledge of libraries
- codellama appears to be lacking at general language tasks

### codellama

```text
Please write the given points in full sentences for a short tech article.

--------

  "Mistral's Coding Task Proficiency: A Comparison with Codellama"

Introduction:
In recent years, the field of natural language processing (NLP) has witnessed
significant advancements in both coding and general language tasks. Among the
various NLP models available in the market, two notable ones are Mistral and
Codellama. In this article, we will delve into their performance in these
domains, with a focus on coding tasks and general language tasks.

Coding Task Proficiency:
Mistral is known for its exceptional ability in coding tasks, particularly in
machine learning-based NLP applications. This is evident from the fact that it
has been used in various projects, such as text classification, sentiment
analysis, and named entity recognition. Codellama, on the other hand, appears
to be equally proficient in coding tasks, with a reputation for its expertise
in Python programming language.

General Language Tasks:
When it comes to general language tasks, Mistral is said to have limitations.
It can struggle with complex grammar and syntax, often producing errors or
unintelligible responses. This has led some users to question the model's
ability to comprehend and generate coherent text in certain contexts. On the
other hand, Codellama appears to excel in these areas, with a reputation for
its fluency and clarity in general language tasks.

Conclusion:
Both Mistral and Codellama have shown promising results in their respective
domains. However, when it comes to coding tasks, Mistral seems to be more
suited to the task. Its expertise in machine learning-based NLP applications
makes it an ideal choice for automating repetitive tasks or performing complex
computations.

In general language tasks, Codellama appears to be more effective, with a
reputation for its fluency and clarity. However, Mistral's limitations in these
areas should not be overlooked, as they may prove useful in certain contexts.
Ultimately, the choice of model depends on the specific requirements of the
project at hand.

References:
1. "Mistral: A Flexible and Scalable Machine Learning Framework for Natural Language Processing" by D. B. M. Al-Abbasi et al. (2020).
2. "Codellama: An Open-Source Python Library for Coding Tasks in NLP" by R. K. M. K. Sasidharan et al. (2020).

--------
```

### mistral

```text
Please write the given points in full sentences for a short tech article.

--------

 Title: "Mistral vs. Codellama: A Comparison of Their Strengths in Tech Tasks"

In the world of technical projects, it is common to encounter two AI language
models with distinct proficiencies - Mistral and Codellama. Let's explore how
they perform in coding tasks and general language tasks.

Mistral demonstrates a commendable aptitude for coding assignments. However, it
is essential to note that Mistral is not only efficient when it comes to
coding; it also displays an acceptable level of competence with general
language tasks.

On the other hand, Codellama excels in coding projects due to its extensive
knowledge base and familiarity with various libraries. Though Codellama
performs well with coding tasks, it is evident that its prowess lies primarily
within this domain. However, it may face challenges when handling more complex
general language tasks.

By understanding each AI's strengths and limitations, we can better allocate
resources and expectations in our technical projects. Stay tuned for further
insights into how Mistral and Codellama can complement your team.

--------
```

### Summary

The result pretty much is spot on and affirms the stated points.

Codellama laments a bunch of nonsense, mixing up facts of the input. It appears
as if it simply "didn't get" the details of the input. It states that
"codellama is equally proficient in coding tasks", hallucinating additional
facts. Additionally, it hallucinates sources for its statements.

Mistral follows the given points very nicely.

## Verdict

I switched to mistral as language model. There might be better coding results
possible with codellama in some cases, but for pretty much _all_ tasks in my
workflow, with the current amount of data I have, mistral gives me better
results consistently.
