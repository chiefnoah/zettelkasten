---
date: 2021-07-14T13:38
tags:
  - blog/published
---

![Keyboard](static/chris-j-davis-PT_9ux0j-x4-unsplash.jpg){.ui .medium .centered .image}

**Additionally posted to:**

* [DEV](https://dev.to/chiefnoah/embed-lua-config-in-your-init-vim-54l1)

# Embed Lua config in your init.vim

With the release of [Neovim 0.5.0](https://neovim.io/), Lua as a first-class
embedded config and plugin language support is now available for general use.

I've started to see Lua plugins pop up more and more, and often their config
documentation is also in Lua. This isn't a problem if you're using an
`init.lua`, but what if you're still using `init.vim`?

Well it turns out embedding Lua config in `init.vim` is pretty easy, just do the
following:

```vimscript
lua << EOS
-- Lua config here
...
EOS
```

That's it!

If you want to see an example of this, check out my
[`init.vim`](https://git.sr.ht/~chiefnoah/dotfiles/tree/nvim-lsp/item/dotfiles/config/nvim/init.vim)!

---

Thanks for reading! If you have any questions or comments, shoot me an email at 
[noah@packetlost.dev](mailto:noah@packetlost.dev) or 
<a href="https://twitter.com/intenttweet?screen_name=chiefnoah13&ref_src=twsrc%5Etfw" class="twitter-mention-button" data-show-count="false">Tweet to @chiefnoah13</a><script async src="https://platform.twitter.com/widgets.js">
