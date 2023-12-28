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

````text
--------

  Here is an example of how to write a C function that calculates the Unix
  timestamp for a given date:
```
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
```
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
```
long int timestamp = get_timestamp(2018, 3, 24);
printf("Timestamp is %ld\n", timestamp);
```
This would print out the Unix timestamp for March 24, 2018.

--------
````

Not ideal, while the code itself is not the real issue.

Asking for more insights made the model run out of VRAM, as the context became
too big.

```text
--------

What is happening here?
```

Boom!


```text

Dec 28 11:56:40 t14s systemd-llama[104760]: Llama.generate: prefix-match hit
Dec 28 11:56:43 t14s systemd-llama[104760]: INFO:     10.0.2.100:57378 - "POST /v1/chat/completions HTTP/1.1" 200 OK
Dec 28 11:57:01 t14s systemd-llama[104760]:
Dec 28 11:57:01 t14s systemd-llama[104760]: llama_print_timings:        load time =     499.81 ms
Dec 28 11:57:01 t14s systemd-llama[104760]: llama_print_timings:      sample time =      61.58 ms /   232 runs   (    0.27 ms per token,  3767.58 tokens per second)
Dec 28 11:57:01 t14s systemd-llama[104760]: llama_print_timings: prompt eval time =    1895.74 ms /   516 tokens (    3.67 ms per token,   272.19 tokens per second)
Dec 28 11:57:01 t14s systemd-llama[104760]: llama_print_timings:        eval time =   17201.30 ms /   231 runs   (   74.46 ms per token,    13.43 tokens per second)
Dec 28 11:57:01 t14s systemd-llama[104760]: llama_print_timings:       total time =   20716.20 ms
Dec 28 11:57:20 t14s systemd-llama[104760]: Llama.generate: prefix-match hit
Dec 28 11:57:22 t14s systemd-llama[104760]:
Dec 28 11:57:22 t14s systemd-llama[104760]: CUDA error 2 at /tmp/pip-install-t_a8l9rk/llama-cpp-python_7eb25cc0ec2440f5a05d50e40f7a3c6d/vendor/llama.cpp/ggml-cuda.cu:6589: out of memory
Dec 28 11:57:22 t14s systemd-llama[104760]: current device: 0
Dec 28 11:57:22 t14s systemd-llama[104760]: GGML_ASSERT: /tmp/pip-install-t_a8l9rk/llama-cpp-python_7eb25cc0ec2440f5a05d50e40f7a3c6d/vendor/llama.cpp/ggml-cuda.cu:6589: !"CUDA error"
Dec 28 11:57:22 t14s systemd-llama[104760]: Aborted
Dec 28 11:57:22 t14s podman[109435]: 2023-12-28 11:57:22.850792922 +0100 CET m=+0.013001423 container died 9978da008aecdd682a553daf3fe0baaa4e90caf89c7d50931eb7ef55602463de (image=ghcr.io/gierdo/dotfiles/llama-cpp-python-server-rocm:0.2.23-11.0.2, name=systemd-llama, org.opencontainers.image.ref.name=ubuntu, org.opencontainers.image.version=22.04, PODMAN_SYSTEMD_UNIT=llama.service, io.buildah.version=1.32.0)
Dec 28 11:57:22 t14s podman[109435]: 2023-12-28 11:57:22.865602338 +0100 CET m=+0.027810829 container remove 9978da008aecdd682a553daf3fe0baaa4e90caf89c7d50931eb7ef55602463de (image=ghcr.io/gierdo/dotfiles/llama-cpp-python-server-rocm:0.2.23-11.0.2, name=systemd-llama, PODMAN_SYSTEMD_UNIT=llama.service, io.buildah.version=1.32.0, org.opencontainers.image.ref.name=ubuntu, org.opencontainers.image.version=22.04)
Dec 28 11:57:22 t14s systemd[2093]: llama.service: Main process exited, code=exited, status=134/n/a
```

Not ideal, and a blocker for the rest of the tasks.

### mistral

```` text
Please create a c function for an embedded platform (no stl or time.h, no
POSIX), which returns the unix timestamp for a given time.
--------

```
#include <stdint.h>

// Helper function to check if a year is a leap year
int isLeapYear(uint32_t year) {
  return (year % 4 == 0 && (year % 100 != 0 || year % 400 == 0));
}

// Function to calculate the number of seconds since January 1, 1970 (Unix
// epoch) for a given year, month, day, hour, minute, and second
uint64_t getEmbeddedUnixTimeFromDate(int year, int month, int day, int hour,
                                     int minute, int second) {
  // Define constants
  const uint64_t JANUARY_1_1970 = 0;  // January 1st, 1970
  const uint64_t ONE_DAY = (24 * 60UL * 60UL * 60UL * 1000UL *
                            365UL);  // Number of seconds in one day
  const uint64_t LEAP_YEARS = ((uint64_t)(year - 1970) / 4) -
                              ((uint64_t)(year / 100) / 4) +
                              ((year % 100 == 0 && (year % 400 != 0))
                                   ? 1
                                   : 0);  // Number of leap years since 1970
  const uint64_t SECONDS_SINCE_JANUARY_1_1970 =
      JANUARY_1_1970 +
      (ONE_DAY * (31UL + ((month - 1) % 12UL) + (isLeapYear(year) ? 29 : 28)));
  const uint64_t HOURS_SINCE_MIDNIGHT =
      hour + minute / 60.0F + second / 60.0F / 60.0F;

  // Calculate and return the number of seconds since January 1, 1970
  return SECONDS_SINCE_JANUARY_1_1970 + HOURS_SINCE_MIDNIGHT * 3600.0F;
}
```
--------

What is happening here?

--------

 This code provides two functions written in C programming language. The first
 function, `isLeapYear`, checks if a given year is a leap year or not based on
 the rules of leap year calculation. A year is considered a leap year if it is
 divisible by 4 but not divisible by 100, except if it is also divisible by
 400.

The second function, `getEmbeddedUnixTimeFromDate`, calculates the number of
seconds since January 1, 1970 (Unix epoch) for a given date, consisting of
year, month, day, hour, minute, and second. The function first determines how
many leap years have occurred since 1970 using the `isLeapYear` helper function
and then calculates the total number of seconds from January 1, 1970, up to the
provided date and time.

--------

What are potential problems with the given code?

--------

 The provided C code is relatively simple and works as intended for a specific
 use case. However, there are some limitations and potential issues that should
 be considered:

1. Validation of input: The current implementation does not provide any
   validation or error handling for invalid inputs like non-existent months,
   incorrectly formatted year values, or out-of-range values for month, day,
   hour, minute, or second. It is recommended to validate the input data and
   return an error code or throw an exception if the input is invalid.

2. Lack of support for leap seconds: The given `getEmbeddedUnixTimeFromDate`
   function does not account for leap seconds, which are occasionally added to
   Coordinated Universal Time (UTC) to maintain atomic clock synchronization
   with Earth's rotation. Leap seconds are added approximately every 18 months
   and should be taken into consideration when working with time on a precise
   level.

3. Inefficient calculation: The given implementation involves some redundant
   calculations, such as converting minutes and seconds to hours for the
   `HOURS_SINCE_MIDNIGHT` constant. This can lead to inefficiencies when
   dealing with large quantities of data or complex applications where
   performance is critical.

4. Lack of portability: The provided code might not be portable between
   different platforms, especially if you are using non-standard functions like
   `isLeapYear`. Additionally, the constant values used (e.g., number of
   seconds in a day) may not be consistent across various compilers and
   systems, so it is recommended to use predefined constants or functions
   provided by standard libraries whenever possible.

5. Limited functionality: The given code does not provide any additional
   functionality like time zone support, date arithmetic, or other advanced
   features that can be useful in real-world applications. To extend the
   functionality, consider using a well-established library like
   Boost.Date_Time, Glib, or GNU C Library's `localtime` and `gmtime` functions
   for handling dates and times more efficiently and accurately while providing
   extensive functionalities.

To mitigate these issues, consider using well-tested libraries that offer
built-in functions for handling time and date calculations or validating input
data, such as Boost.Date_Time, Glib, or GNU C Library's `localtime` and
`gmtime` functions. These libraries provide robust error handling, portability,
and efficient implementations, allowing you to focus on the core functionality
of your application without having to worry about low-level time and date
calculations.


--------

Please give me a well-documented version of this code. Please follow best
practices and do not comment overly verbosely.

--------

 ```c
#include <stdint.h> // Include standard integer types header

// Helper function to check if a year is a leap year
bool isLeapYear(uint32_t year) {
  // A year is a leap year if it's divisible by 4, but not by 100, unless it's also divisible by 400
  return (year % 4 == 0 && (year % 100 != 0 || year % 400 == 0));
}

// Function to calculate the number of seconds since January 1, 1970 (Unix epoch) for a given date and time
uint64_t getEmbeddedUnixTimeFromDate(int year, int month, int day, int hour,
                                    int minute, int second) {
  // Constants:
  static const uint64_t JANUARY_1_1970 = 0;  // January 1st, 1970 (Unix epoch)
  static const uint64_t ONE_DAY = (24UL * 60UL * 60UL * 60UL * 1000UL * 365UL); // Number of seconds in one day
  static const int LEAP_YEARS = 0;           // Initialize leap years constant

  // If the year is not a leap year, calculate the number of leap years since 1970 by subtracting the number of non-leap years (common years)
  if (!isLeapYear(year)) {
    LEAP_YEARS = ((uint64_t)(year - 1970) / 4) - ((uint64_t)(year / 100) / 4);
  }

  // Else, if the year is a leap year and divisible by 400, add an extra leap year to account for 1900 being a leap year
  else if (year % 400 == 0) {
    ++LEAP_YEARS;
  }

  // Calculate the total number of seconds since January 1, 1970
  const uint64_t SECONDS_SINCE_JANUARY_1_1970 = JANUARY_1_1970 +
                                               (ONE_DAY *
                                                (31UL + ((month - 1) % 12UL) +
                                                        (isLeapYear(year) ? 29 : 28)));

  // Add the number of seconds from the given date and time to the Unix epoch
  const uint64_t HOURS_SINCE_MIDNIGHT = hour + minute / 60.0F + second / 60.0F / 60.0F;
  const uint64_t final_time = SECONDS_SINCE_JANUARY_1_1970 + HOURS_SINCE_MIDNIGHT * 3600.0F + LEAP_YEARS * ONE_DAY;

  return final_time;
}
```

This version of the code is well documented, follows best practices, and is
easily readable while providing essential context for each part of the
function. The constants are defined with clear variable names, and each step in
the calculations is explained concisely. Additionally, I've used `const` and
`static` keywords where appropriate to minimize the impact on performance and
memory usage.

--------
````

Mistral seems to fully understand the given task and solves it as requested.


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
