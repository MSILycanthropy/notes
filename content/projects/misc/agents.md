---
title: Agents
---

I don't love AI, but I also don't hate it. I think it has its uses. It's nice to do automated tasks that I don't want to do. But it's not quite smart enough to take my job, for now. So, I generally tend to have it do a bunch of random bullshit for me. Stuff like finding relevant lines in code, converting formats, giving me regex, etc. And I normally use [Claude](http://claude.ai) for that.

I'll kinda just start 1 off chats, make it do something for me and then leave it. But, today I found a cool little Ruby library for messing with AI and figured I'd fuck around.

## Roast

That library is from our overlords at Shopify. It's [Shopify/roast](https://github.com/Shopify/roast), a nice Ruby DSL for AI workflows. Finally, a way to structure my menial tasks I wanna make AI do.

The pitch is pretty simple: you chain together "cogs" (their word for building blocks) that can run shell commands, execute Ruby code, talk to LLMs, or even spin up local coding agents with filesystem access. You write declarative Ruby, and it handles the plumbing of passing outputs between steps.

## CLI

So, I coded up a quick little CLI [MSILycanthropy/agents](https://github.com/MSILycanthropy/agents) to create some agents. To dogfood it I forced Claude Code to do a little bit of it at gunpoint.

The whole thing is pretty lightweight. Clone it, run the install script, and you get a global `agents` command. From there you can:

- **`agents list`** — see what agents are available
- **`agents commit`** — generate a commit message from your staged changes
- **`agents convert myfile.md -- json`** — convert files between formats
- **`agents regex -- "email addresses"`** — have it spit out a regex for whatever you describe
- **`agents tags README.md`** — auto-suggest tags for a file

There's also debug and verbose modes if you want to see what's actually happening under the hood.

Nothing groundbreaking, but it's nice having these little utilities in one place instead of opening a new Claude chat every time I need a regex or a commit message. And since it's all built on Roast, adding new agents is just writing a bit of Ruby.
