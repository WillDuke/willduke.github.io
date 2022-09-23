---
title: "The Trouble with Notebooks"
date: 2022-07-28T00:00:00-00:00
draft: false
categories: 
    - python
tags: 
    - python
    - notebook
    - jupyter
---

Notebooks are exceptionally popular among data scientists. So much so that many of my colleagues work largely in JupyterHub or Sagemaker. Notebooks provide a convenient enviroment for quick prototyping, and they feel natural for data scientists that (like me) don't have a formal computer science background. That said, I'm convinced that we are using them in many cases where they aren't worth the downsides, and even that the popular wisdom that they are a good place to get started quickly has it quite backwards.

Well, let me first make sure to give notebooks a fair shake. There are a ton of cool things you can do with notebooks that I think are great:

* **Avoiding (initial) packaging headaches** with a pre-configured environment is a huge boon to data scientists who don't want to deal with virtual environments and dependencies when doing one-off analyses.
* **Writing narrative-driven code** with the literate style of programming made possible with code and markdown cells plus inline output makes for great reports.
* **Developing visualizations** is so natural in a notebook when you want to iteratively refine a plot or series of plots for your eventual output. (Data scientists love this because it reminds us of the dedicated plot window in RStudio...[Spyder, anyone?](https://www.spyder-ide.org/)).
* **Demoing code for others** can be easy and expressive in a notebook, when you can prepare each cell to return intermediate results and walk someone else through your code.
* **Fancying up your documentation** using tools like [nbsphinx](https://docs.readthedocs.io/en/stable/guides/jupyter.html) which render notebooks right into your Sphinx docs, helping others get started with your project.

All that aside, here are some downsides to watch out for:

### It's Hard to Make Code Modular

Ideally, the source code for a project should be broken into logical pieces -- functions, classes, and modules contributing distinct features to a project. This value is built into many programming tools: Linters now warn on long methods, and some projects even reject commits on the basis of [cyclomatic complexity](https://github.com/pycqa/mccabe) via CI checks.

But fundamentally, the pattern of working in a notebook makes this process of decomposition unnatural. Nothing queues the user that there is a difference between definition and execution code, so function definitions often get mixed in with parts of the analysis. This makes it difficult for someone reading the code to understand the context of its execution.

What's more, the individual cells feel a little like functions themselves, which makes it easy to write a lot of top-level code that is then run almost like a script to load or modify an object that is used elsewhere. I think this contributes to the state of many notebooks I see which have almost no functions defined.

These patterns are forgivable given how unwieldy refactoring is in a notebook. Copying code between cells involves a lot of scrolling -- and all too quickly the notebook gets messy and large enough that most of the relevant code is not visible at the same time. At least for me, I need to be looking at the code as I reason about it; otherwise, I seem to get caught in an endless loop of scrolling up to remember exactly how that helper method was defined, scrolling down to use it, making an error, and scrolling up again.

### It's Tough to Handle Implicit State

Once a notebook gets a little dirty, it becomes easy to get into a state that is difficult to reproduce. I seem to find it both incredibly tempting to run cells out of order, and then incredibly difficult to reconstruct the state of the notebook later. I think to myself,

> Oh, I'll just fix that bug in an earlier cell and re-run it. *(repeats a few times)*  
> OK, I think I'm in a good place for the day *(shuts down kernel)*  
> ***chaos ensues***

It's far too easy to create circumstances where code that appears to be perfectly valid raises an error because of some unusual hidden state.

The ability to freely run cells out of order, while convenient, breaks our convenant with the machine that - barring the explicit introduction of concurrency -- things will happen (or have happened) in a predictable linear order. I can look at my screen and see two cells that define a logical set of steps and then get a confusing error on running the latter cell because a variable was silenty overwritten when I ran a third cell I thought was unrelated.[^1]

### Reproducing Results can be Tricky

The output of a notebook depends on the kernel that was used to run it, and different individuals may not have access to or have set up the same environment. Sometimes, the notebooks themselves contain `pip` magic that install new packages or makes other environment changes. This can make it very difficult to reproduce the set of packages that were used to generate the results. Notebooks give the appearance of being standalone files, so they are frequently exchanged without bringing along enough information to reproduce the initial kernel state.

### Tooling is Limited

Notebooks don't have great support for many useful productivity tools:

* Notebooks accessed through a web interface lack autocomplete, built-in rollover suggestions, method extraction, debugging, etc.
* Without workarounds, diffs in source control are essentially useless
* Testing frameworks, linters, autoformatters, and other code quality tools have limited support

Without these tools, it's much more difficult to maintain code style and quality standards and maintain confidence that code is working as intended.

### Deploying

Eventually, we'd like to deploy our models -- at which point we'll want all of the extra tooling and a complete description of our dependencies anyway!

### Wrapping Up

The trouble with notebooks that don't have separate definition and execution code, modularity, or a necessarily linear execution order is, put simply, *that they usually don't run*.

How many times have you received a notebook of any appreciable length, hit run all, and made it through to the last cell?

It's happened a few blessed times for me, but mostly I'm quickly swearing, trying to figure out what went wrong.

I think that notebooks impose such a cost on reproducibility, collaboration, and even code correctness that they are just not worth the convenience outside demos, informal experimenting, and results sharing.

[^1]: For especially good examples of this phenomenon and more issues with notebooks, check out Joel Grus' entertaining talk *[I Don't Like Notebooks](https://www.youtube.com/watch?v=7jiPeIFXb6U)*. The memes alone make it worth a watch.
