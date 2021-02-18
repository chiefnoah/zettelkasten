---
date: 2020-08-05T12:00
tags:
  - blog/published
---

# The Terminal Jockey's Toolbelt

How much do you think software #development has changed over the last 20 years? Many would say quite a bit. But, my opinion is that fundamentally not much has really changed. Many of us still use the tried and true method of telling a computer what to do: the terminal. While the #terminal has changed it's appearance a bit over the years, it's still fundamentally the same tool it was 50 years ago. But the terminal is really only the window, and the tools we use through it *have* changed over the years. At least for some of us. As someone who spends most of his day bashing away at my clicky mechanical keyboard, I'd like to share some of the modern tools that I use to do my job, many of which replace commonly installed *nix tools. 

<!--more-->

I'm always looking to improve and integrate new tools into my workflow. If you have any suggestions, feel free to email me at the address at the bottom of this post or message me on [Twitter](https://twitter.com/chiefnoah13).

---

## 🔥Hot🔥 Reload All the Things

[entr(1)](http://eradman.com/entrproject/)

> Edit → Compile → Test → Edit → Compile → Test → Edit → Compile → Test

Have you ever found yourself in a loop of editing a file, executing/compiling it over and over and over again? Well here's the solution to your *home row →* ⬆️ RSI. 

Well repeat no more! Enter `entr`, stage left. `entr` is a fairly simple command line utility that will execute a command whenever a file or list of files change. When combined with a utility like `find` or `fd` it becomes insanely powerful.

Some examples of what I use it for:

```bash
fd ".+\.py" | entr python -m pytest some_test.py
fd -g "*.go" | entr -r go run cmd/server.go
```

My day job is primarily python, so a common use-case for me is in a `:term` using `neovim` with a similar command to run tests automatically after each file save, and similar for compiled languages such as Go.

Another common one is when developing a web server application. For example, in Go I use a command similar to the following:

```bash
fd -g "*.go" | entr -r sh -c "go build cmd/server.go -o server && ./server"
```

This rebuilds and runs the server after every file change. The `-r` flag sends a SIGINT to the process when a file change is detected, and then re-executes the command. Pretty neat, right?

It's worth calling out that this is a common feature in many IDEs. As an occasional user of PyCharm, I've used the built-in equivalent to achieve the same result. However, I prefer to work in vim, so this fills that role for me. 

## What was that flag again?

[fd(1)](https://github.com/sharkdp/fd)

`find` has been a long-time tool for *nix users. But for someone who hasn't gotten used to the esoteric syntax of `find`, or is looking for something that requires typing a bit less (among many other really nice features), look no further than `fd`. You may have noticed I use `fd` in my previous example, `entr`.

I typically prefer glob patterns to regex for quick searches, so most of the time I add a `-g` to my commands to enable glob mode.

One of the areas that `fd` really shines is an improved templating syntax for the `--exec` / `-e` flag. The main GitHub readme has some pretty solid examples, so be sure to check out their [README](https://github.com/sharkdp/fd/blob/master/README.md).

## The Silver Searcher

[ag(1)](https://github.com/ggreer/the_silver_searcher)

My life was changed the day I learned aboug `grep -r`. Recursively searching for text in a codebase is easily one my most common daily tasks. For my day job, we have... a *lot* of files, some rather large. The Silever Searcher, or just `ag` (the atomic symbol for Silver 🙂) is a simple replacement for `grep -r` that's more performant and has nice ANSI color coding in the terminal. The syntax is pretty similar to `grep` so it makes for a good drop-in replacement to speed up your workflow by those *crucial* few milliseconds.

![](https://i.snap.as/U2HscOd.png)

## All things old are new again, and with Lua

[nvim(1)](https://neovim.io/)

`nvim` is a fork of the infamous `vim` project. Much of the project has focused on cleaning up the codebase, adding async support (which was subsequently added to `vim` for version 8), and Lua as a first-class language. The Lua API in particular allows for more robust and easy to maintain plugins (in theory). That being said, I'm not big on diving deep into customizing my editor, though I've made several shortcuts and rely on a handful of plugins. I generally tend to focus on memorizing the default keybindings so when moving from system to system I don't have to unlearn or mess up text entry. Check out my [dotfiles](https://git.sr.ht/~chiefnoah/dotfiles/tree/master/dotfiles/config/nvim/init.vim)!

Regardless of which editor you use on the daily, it's worth learning the basics of `vim`, it's available on nearly every Linux or Unix system and is particularly fantastic at making quick edits to config files.

![](https://i.snap.as/JyqcvJ8.png)

## Email for the 20th century

[aerc(1)](https://aerc-mail.org/)

`aerc` is one of [Drew DeVault](https://drewdevault.com/)'s projects, and was originally built to suit his particular workflow. Despite this, it's actually a really nice modern implementation of an IMAP4/SMTP/POP3 client for the terminal (provided you primarily use `text/plain`). It's far more intuitive than `mutt` or other alternatives that I've tried, and it's **fast**. Well, usually. I've noticed some minor performance issues with a high latency connection to the mail server, but overall the experience is fantastic and it's my client of choice for email. 

![](https://i.snap.as/GjcsDLV.png)

## Markdown rendering... in the terminal!?

[mdr(1)](https://github.com/MichaelMure/mdr)

Have you ever been working on a `README.md` and somehow forgotten that you need a double newline for it to actually render properly in markdown? I know I have way more than is reasonable for someone in my profession. Well to help with markdown rendering woes, `mdr` renders markdown with color highlighting and rich text formatting! It even tries to render images!

Given that not all of markdown's output format (typically HTML) is compatible with rendering to a terminal, it does make some compromises, but I've found it's fine 99% of the time.

![](https://i.snap.as/1KPbMlA.png)

For comparison on what this looks like rendered to HTML, check out that project's [README.md](https://git.sr.ht/~chiefnoah/origin/tree/master/README.md).

## Honorable Mentions

Some other tools that I like to use when appropriate, but I didn't feel like including in the main list:

- [`redo(1)`](https://github.com/apenwarr/redo) - `make` but much, *much* better
    * I prefer this to `make` as it lacks some of the strange formatting quirks of the Makefile, and it is generally easier to work with and understand
- [`tmux(1)`](https://github.com/tmux/tmux/wiki) - terminal multiplexing, who needs a desktop?
- [`tldr(1)`](https://tldr.sh/) - `man` pages quick reference
    * I don't actually use this one as much yet, I just haven't integrated it into my workflow, but I appreciate the project and likely will get to it eventually
- [`ripgrep`/`rg(1)`](https://github.com/BurntSushi/ripgrep) - similar to `ag(1)`, a faster more modern `grep -r` command
    * This one was added based on feedback. I don't use it, but many seem to prefer it

---

Thanks for reading! If you have any questions or comments, shoot me an email at
[noah@packetlost.dev](mailto:noah@packetlost.dev) or <a
href="https://twitter.com/intent/tweet?screen_name=chiefnoah13&ref_src=twsrc%5Etfw"
class="twitter-mention-button" data-show-count="false">Tweet to
@chiefnoah13</a><script async src="https://platform.twitter.com/widgets.js"
charset="utf-8"></script>
