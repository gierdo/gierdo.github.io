---
title: Local AI assistant in vim - Performance evaluation
categories:
  - programming
  - linux
tags:
  - vim
  - openai
  - ai
  - containers
  - llama
  - ROCm
  - AMD
---

# Update: Local AI assistant in vim with codellama - performance evaluation

You might have read my previous posts about [using a local codellama for a vim
coding assistant]({% link _posts/2023-11-24-llama-vim.md %}) and setting it up
to use [my laptop's GPU for inference]({% link
_posts/2023-12-16-llama-vim-igpu-amd.md %}).

The system now has many degrees of freedom that need a bit of systematic
evaluation.

## The Problem

I run a `codellama` as local assistant in my editor using `llama.cpp`. With the
setup, there are now these fundamental degrees of freedom that have an impact
on the performance characteristics of the inference:

- Model size
  - codellama-7b-instruct.Q4_K_M.gguf
  - codellama-13b-instruct.Q4_K_M.gguf
- Type of acceleration
  - CPU-only with OpenBLAS
  - GPU-assisted with ROCm
- Configuration details
  - Context window size
  - Processing batch size
  - Number of GPU-offloaded layers

The affected characteristics are mainly

- Resource consumption
  - Memory (RAM, VRAM)
  - CPU
  - Battery
- Speed (tokens per second)
- Answer quality

I want to find a good compromise that allows me to work with it, ideally also
on battery.

## The test

I am not really interested in a full scientific evaluation or benchmark, but I
still want to find a sensible compromise. So, here is what I do:

### Environment

- Lenovo Thinkpad t14s gen 4
- _AMD Ryzen 7 PRO 7840U w/ Radeon 780M Graphics_
- 32G of RAM in total, split into 24G RAM and 8G VRAM
- Test execution on battery power

### Benchmark

I recently started digging into the code of
[pandoc](https://github.com/jgm/pandoc), which is written in `haskell`. I put
this function, part of the `Docx` writer, into the context.

```haskell
withDirection :: PandocMonad m => WS m a -> WS m a
withDirection x = do
  isRTL <- asks envRTL
  paraProps <- asks envParaProperties
  textProps <- asks envTextProperties
  -- We want to clean all bidirection (bidi) and right-to-left (rtl)
  -- properties from the props first. This is because we don't want
  -- them to stack up.
  let paraProps' = filter (\e -> (qName . elName) e /= "bidi") (otherElements paraProps)
      textProps' = filter (\e -> (qName . elName) e /= "rtl") (otherElements textProps)
      paraStyle = styleElement paraProps
      textStyle = styleElement textProps
  if isRTL
    -- if we are going right-to-left, we (re?)add the properties.
    then flip local x $
         \env -> env { envParaProperties = EnvProps paraStyle $ mknode "w:bidi" [] () : paraProps'
                     , envTextProperties = EnvProps textStyle $ mknode "w:rtl" [] () : textProps'
                     }
    else flip local x $ \env -> env { envParaProperties = EnvProps paraStyle paraProps'
                                    , envTextProperties = EnvProps textStyle textProps'
```

Then I ask codellama to explain it to me.

```text
Can you explain this to me?
```

I will collect the resource consumption and detailed characteristics, as well
as the given answer, for a set of different configurations.

## Results

### codellama-7b-instruct

The first set uses the 7b parameters model of codellama in the `Q4_K_M`
variant, as available on
[huggingface](https://huggingface.co/TheBloke/CodeLlama-7B-Instruct-GGUF/resolve/main/codellama-7b-instruct.Q4_K_M.gguf).

#### GPU inference, maximum layers

The model has a small memory footprint, which allows me to run _all_ layers on
the GPU without running out of VRAM.

##### Answer

```text
Can you explain this to me?

--------

Sure! This code defines a higher-order function `withDirection` that takes a
`WS m a` value as input and returns another `WS m a` value. The function
performs some operations on the environment properties of the document,
specifically removing any "bidi" (bidirectional) or "rtl" (right-to-left)
elements from the existing properties and then adding them back in if
necessary.

The code is using the `asks` function to retrieve the current environment
properties and a few other variables from the context, such as the direction of
the text (`isRTL`) and the styles for the paragraph and text elements
(`paraStyle`, `textStyle`). It then filters out any "bidi" or "rtl" elements
from the existing properties using a guard expression (the `if` statement) and
adds them back in if necessary.

Finally, the function uses the `local` function to modify the environment
properties of the document for the duration of the provided action `x`. The
modified environment is then used to run the rest of the code in the context of
that new environment.

So, in short, this function is modifying the bidirectional and right-to-left
properties of the document based on a certain condition, which I assume is
related to the direction of text flow in the document.

--------
```

##### Performance

```text
llama_print_timings:        load time =    2123.07 ms
llama_print_timings:      sample time =      39.34 ms /   280 runs   (    0.14 ms per token,  7117.26 tokens per second)
llama_print_timings: prompt eval time =   33382.56 ms /    16 tokens ( 2086.41 ms per token,     0.48 tokens per second)
llama_print_timings:        eval time =   71125.86 ms /   279 runs   (  254.93 ms per token,     3.92 tokens per second)
llama_print_timings:       total time =  105680.76 ms
```

##### Resource consumption

Everything happens in the GPU, the fans remain silent, no CPU spike whatsoever.

![Resources](/assets/images/2023-12-17-llama-vim-igpu-amd-performance/7b-gpu.png)

```text
Memory (total): 6.3G
VRAM: 6873M / 8095M VRAM  84.90%
```

#### CPU only

##### Result

```text
Can you explain this to me?

--------

Certainly! This code defines a function `withDirection` that takes an input
value of type `WS m a` and returns a new value of the same type. The function
modifies the environment in which the input value is evaluated, by adding or
removing certain properties (represented as XML elements) from the
`envParaProperties` and `envTextProperties` fields of the environment.

The function first checks whether the environment indicates that we are going
right-to-left (`isRTL`). If so, it adds the properties back to the environment.
If not, it simply removes them from the environment.

The function then modifies the `envParaProperties` and `envTextProperties`
fields by filtering out certain elements from their lists of properties
(represented as XML nodes) and adding a new node for the right-to-left property
if we are going right-to-left. The `styleElement` function is used to create an
element representing the style of the text or paragraph, which is then added to
the list of properties.

The resulting modified environment is then used to evaluate the input value
using the `local` function, which creates a new environment that contains the
modified values.

--------
```

##### Performance

```text
llama_print_timings:        load time =   30617.12 ms
llama_print_timings:      sample time =      39.47 ms /   256 runs   (    0.15 ms per token,  6485.28 tokens per second)
llama_print_timings: prompt eval time =   30616.33 ms /   380 tokens (   80.57 ms per token,    12.41 tokens per second)
llama_print_timings:        eval time =   98087.64 ms /   255 runs   (  384.66 ms per token,     2.60 tokens per second)
llama_print_timings:       total time =  130449.74 ms
```

##### Resource consumption

CPU goes to around 25% during the prompt evaluation and 50%+ during the
generation.

![Resources](/assets/images/2023-12-17-llama-vim-igpu-amd-performance/7b-cpu.png)

```text
Memory (total): 5.1G
VRAM: N/A
```

#### Bonus round: With AC power (non-powersave GPU profile)

As stated in the environment section, I want to explicitly compare the mobile
usecase with  powersaving profiles enabled.

Switchint off powersaving configurations and enabling the full power of the GPU
gives a huge performance improvement, especially on the pure GPU inference:

##### Answer

```text
Can you explain this to me?

--------

Certainly! The `withDirection` function is a higher-order function that takes
an `WS` monad and returns an equivalent `WS` monad that has been modified to
handle bidirectional (bidi) and right-to-left (rtl) text properties.

Here's what the code does:

1. It checks if we are in a right-to-left context (isRTL).
2. If so, it cleans up any existing bidi or rtl properties by filtering them
   out of the `paraProps` and `textProps`.
3. It creates new sets of properties with the bidi/rtl flags removed from the
   previous `paraProps'` and `textProps'`.
4. It creates a new set of properties that includes only the style elements
   (e.g., paragraph or text styles) for the given context, using the
   `styleElement` function.
5. Finally, it modifies the original `WS` monad to add the new bidi and rtl
   properties, either by adding them to the existing set of properties or
   replacing any existing ones with the new ones.

The purpose of this function is to ensure that bidi and rtl properties are
properly managed when creating Word documents. By cleaning up existing
properties before adding new ones, we avoid having multiple instances of these
flags stack up and cause errors in the final document.

--------
```

##### Performance

```text
llama_print_timings:        load time =    1295.59 ms
llama_print_timings:      sample time =     100.39 ms /   296 runs   (    0.34 ms per token,  2948.50 tokens per second)
llama_print_timings: prompt eval time =    1295.30 ms /   380 tokens (    3.41 ms per token,   293.37 tokens per second)
llama_print_timings:        eval time =   23413.66 ms /   295 runs   (   79.37 ms per token,    12.60 tokens per second)
llama_print_timings:       total time =   26681.01 ms
```

### codellama-13b-instruct

The second set uses the 13b parameters model of codellama in the `Q4_K_M`
variant, as available on
[huggingface](https://huggingface.co/TheBloke/CodeLlama-13B-Instruct-GGUF/resolve/main/codellama-13b-instruct.Q4_K_M.gguf).

#### GPU inference, maximum layers

The larger 13b parameters model does not fit in the 8G VRAM of my integrated
GPU. I can offload a maximum of 30/43 layers to the GPU, or llama will run out
of VRAM during inference.

##### Result

This configuration gave the most verbose answer while testing, but that is not
a systematic property of the model/configuration.
Pretty weird, though

```text
Can you explain this to me?

--------

Sure, I'd be happy to help! This looks like a function that takes a `WS` monad
as input and returns the same `WS` monad with some additional properties. Here
are my interpretations of each part:

1. `withDirection x = do`: This line defines a new function called
   `withDirection` that takes an argument `x` and returns a `WS` monad. The
   `do` keyword is used to introduce a block of code in which the `WS` monad
   can be transformed.
2. `isRTL <- asks envRTL`: This line uses the `asks` function from the
   `ReaderT` monad transformer to extract the value of the `envRTL` field from
   the environment. The `asks` function takes a function that extracts a value
   from the environment, and returns a new `ReaderT` computation that applies
   the given function to the current environment and returns the result. In
   this case, `envRTL` is a function that extracts the value of the `rtl` field
   from the environment, so `asks envRTL` will return the value of the `rtl`
   field.
3. `paraProps <- asks envParaProperties`: This line uses the same pattern as
   the previous line to extract the value of the `envParaProperties` field from
   the environment. The result is a new `WS` monad that returns the value of
   `envParaProperties`.
4. `textProps <- asks envTextProperties`: Similarly, this line extracts the
   value of the `envTextProperties` field from the environment using the same
   pattern as before.
5. `let paraProps' = filter (\e -> (qName . elName) e /= "bidi") (otherElements
   paraProps)`: This line defines a new variable called `paraProps'` that is
   equal to the result of filtering the elements in `paraProps` using a lambda
   function. The filter function takes an element and returns `True` if the
   element should be included in the filtered list, or `False` otherwise. In
   this case, the filter function checks whether the name of the element starts
   with "bidi", and excludes any elements that do not start with "bidi".
6. `textProps' = filter (\e -> (qName . elName) e /= "rtl") (otherElements
   textProps)`: Similarly, this line defines a new variable called `textProps'`
   that is equal to the result of filtering the elements in `textProps` using
   the same lambda function as before.
7. `paraStyle = styleElement paraProps`: This line defines a new variable
   called `paraStyle` that is equal to the result of calling the `styleElement`
   function on `paraProps`. The `styleElement` function takes a list of
   elements and returns a single element with all the styles applied. In this
   case, it applies the styles in `paraProps` to the text.
8. `textStyle = styleElement textProps`: Similarly, this line defines a new
   variable called `textStyle` that is equal to the result of calling the
   `styleElement` function on `textProps`. The resulting element contains all
   the styles from `textProps`, which are applied to the text.
9. `if isRTL then ... else ...`: This line checks whether the value of `isRTL`
   is true or false, and executes one of two different blocks of code depending
   on the result. If `isRTL` is true, then the first block of code is executed,
   which adds new properties to the environment using the `local` function from
   the `ReaderT` monad transformer. Specifically, it adds a property called
   "bidi" and sets its value to the current environment, and also adds a
   property called "rtl" with an empty list as its value.
10. If `isRTL` is false, then the second block of code is executed, which
    removes the properties that were added in the previous step using the
    `local` function. It does this by creating new variables `paraProps'` and
    `textProps'` that exclude any elements with names starting with "bidi" or
    "rtl", respectively.
11. The final line of code is a call to the `flip` function from the
    `Control.Functor` module, which takes a function and applies it to its
    arguments in reverse order. In this case, the function is `local`, and the
    arguments are `x` (the original input) and a new environment with the
    updated properties.

In summary, the `withDirection` function takes an input `WS` monad and returns
a new `WS` monad that contains the same elements as the original, but with
additional properties depending on whether the text is going right-to-left or
not. If the text is going right-to-left, it adds a "bidi" property to the
environment with an empty list as its value, and also adds a "rtl" property
with an empty list as its value. Otherwise, it removes these properties from
the environment using `local`.

--------
```

##### Performance

```text
llama_print_timings:        load time =   10534.74 ms
llama_print_timings:      sample time =     178.59 ms /  1078 runs   (    0.17 ms per token,  6036.31 tokens per second)
llama_print_timings: prompt eval time =   10534.06 ms /   380 tokens (   27.72 ms per token,    36.07 tokens per second)
llama_print_timings:        eval time =  630126.33 ms /  1077 runs   (  585.08 ms per token,     1.71 tokens per second)
llama_print_timings:       total time =  646330.43 ms
```

##### Resource consumption

CPU goes to around 50% right away and stays there.

![Resources](/assets/images/2023-12-17-llama-vim-igpu-amd-performance/13b-gpu.png)

```text
Memory (total): 11.0G
7683M / 8095M VRAM  94.90%
```

#### CPU only

##### Result

It is clear that the 13b parameters model is more capable. It knows
about Haskell's monads, and it recognizes the origin of the code snippet.
It also makes clear that it requires a bit more information to fully explain.

```text
Can you explain this to me?

--------

Sure, I'd be happy to help! This code appears to be related to the `pandoc`
library for working with Markdown and other document formats. It looks like it
is defining a function called `withDirection` that takes a `WS m a` monad and
returns another `WS m a` monad.

The function seems to be concerned with handling bidirectional (bidi) and
right-to-left (rtl) text in a document, and it appears to modify the properties
of paragraphs and text elements within those paragraphs as needed. The code
uses some Pandoc-specific data types and functions, such as `EnvProps`,
`otherElements`, and `styleElement`, which I'm not familiar with.

Without more context or information about what this function is intended to do,
it's difficult for me to provide a more detailed explanation of its purpose or
behavior. If you have any questions or need further clarification, please feel
free to ask!

--------
```

##### Performance

```text
llama_print_timings:        load time =   54705.71 ms
llama_print_timings:      sample time =      36.50 ms /   211 runs   (    0.17 ms per token,  5780.35 tokens per second)
llama_print_timings: prompt eval time =   54704.96 ms /   380 tokens (  143.96 ms per token,     6.95 tokens per second)
llama_print_timings:        eval time =  152508.10 ms /   210 runs   (  726.23 ms per token,     1.38 tokens per second)
llama_print_timings:       total time =  208819.88 ms
```

##### Resource consumption

CPU goes to around 25% during the prompt
evaluation and 50%+ during the generation.

![Resources](/assets/images/2023-12-17-llama-vim-igpu-amd-performance/13b-cpu.png)

```text
Memory (total): 9.2G
VRAM: N/A
```

## Verdict

- No surprise, the 13b model is more knowledgeable than the 7b model
- GPU inference on _my_ setup is around 1.5 times quicker (3.9 t/s vs 2.6 t/s)
  if everything happens on the GPU, which limits the use to the 7b codellama
  model
- For the 13b model, GPU inference on _my_ setup is around 1.2 times quicker
  (1.71 t/s vs 1.38 t/s)
- CPU inference with the 7b model is 1.88 times quicker (2.6 t/s vs 1.38 t/s)
- GPU inference with the 7b model is 2.2 times quicker (3.9 t/s vs 1.71 t/s) on
  _my_ setup. This is mainly due to the fact that only the 7b model can fully
  run on the GPU, the 13b model has to run on CPU and GPU
- GPU inference without powersaving configuration is veeery fast (12.6 t/s vs
  3.9 t/s)

I will most likely run with the _7b model_ on full GPU. The benefits of the
quickest inference, combined with silence and battery consumption outweigh the
quality benefit of the larger model for my typical use.
