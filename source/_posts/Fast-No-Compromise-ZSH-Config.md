---
title: Fast, No-Compromise ZSH Configuration
date: 2020-11-15 13:00:00
tags:
  - configuration
  - shell
  - linux
  - terminal
---

In my experience, having looked at other blog posts and Stack Overflow answers the topic, there seems to be two distinct categories of ZSH users. There are those that like to keep things minimal to have the fastest possible experience, and there are those that like to have a feature-packed shell and who don't mind a slight loss in performance.

Personally, I've given all the most popular shells a try : bash, zsh, fish. The one I stuck with the longest have been zsh and fish. Like most people, my first experience was with bash. Eventually, though, I learned of zsh and its configuration frameworks. They appealed to me because they provided many awesome utilities out of the box!

## Fish

Fish is most certainly the friendliest (hence the name) of all shells in its default configuration. It has autocomplete, syntax highlighting, aliases, all out of the box without having to install any plugins. Its syntax is (in my opinion) more intuitive and easier to use than bash and zsh, mostly because it's not trying to be compatible with either of them.

My biggest problem with fish is that it's quite slow compared to the other options. On average, it takes about 15ms for fish to boot. That's not even a cold start when it has to update its man page completion database. Now, 15ms may seem like an extraordinarily small amount of time to get upset about, but in contrast, bash boots in about 4ms. Fish is almost four times slower.

## ZSH Configuration Frameworks

Since fish was too slow for me, I decided to turn to ZSH, because it offers more features than bash and is more modern, but its startup time is just as fast as bash in its stock configuration. Though, to get close to the level of functionality offered by fish with zsh, I thought I'd have to use a configuration framework.

By far, the most well-known and popular framework is `oh-my-zsh`. It's really a great framework and it's extremely customizable. With `oh-my-zsh`, there's a plugin for just about everything.

Then there's `prezto`, which is a bit less well-known, but probably just as feature-packed as `oh-my-zsh`. Though, its main selling point is that it's fast, which it is!

Overall, both `oh-my-zsh` and `prezto` offer a great user experience, but because I was looking to have the fastest shell possible while also having the most features, I decided to go with `prezto`.

## Prezto

In its default configuration, prezto is indeed very fast (except on a colt start where it needs to re-build some caches). It performs about as well as bash for startup times, which is a good start! With some plugins enabled, prezto boots in roughly 7ms, which is _very_ good, but still almost twice as slow as bash.

In the end, I still decided to swap out prezto for an even more minimal setup, and better prompt customizability. Though, I still think prezto is a great framework and it delivers on its promise of being fast.

## Something More Custom

Recently, I learned about [starship.rs](https://starship.rs/), which is a "minimal, blazing-fast, and infinitely customizable prompt for any shell". Being a very happy user or Rust, I decided to check it out. I was blown away by the ease of use and the speed of this prompt, so I decided that I had to try it for myself. I had also been casually looking for a way to easily create a customized prompt for my shell, which didn't seem trivial with prezto.

The desire to try starship prompted me to ask myself if I really needed all the plugins that prezto offered. Realistically, I was mostly only using the ssh plugin and the prompt plugin. After realizing this, I decided to create my own, minimal configuration for zsh without any framework.

Doing this was far easier than I initially imagined, so let's walk through the process!

First, I installed starship, then I also added [zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions) and [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting). After having installed everything, I created some aliases, copied the `ssh-agent` script from prezto ([here](https://github.com/sorin-ionescu/prezto/tree/master/modules/ssh)) and sourced everything in `.zshrc`.

After some very minimal configuration to starship, I was done. My prompt now has all the information I want, no more, no less, and zsh boots in about 4.5ms, which is just a tiny bit slower than bash. Though, considering bash is already so fast to boot, being only half a millisecond slower is very satisfactory.

For an example of the complete setup, see [this](https://github.com/marier-nico/dotfiles/commit/fa7ca4845c7617e1e703c1ee3ce63235eda08337) commit in my dotfiles.

## Conclusion

Overall, I'm very happy to have gone on this journey to test out different shells with different configuration options and settings. I think my quest to find the perfect shell has finally concluded. Good luck on your own shell adventures, it's a fun world to explore!

### Notes

If you'd like to time your own shell boot times in the same way that I've done it, you can use the following command : `python -m timeit "__import__('os').system('zsh -c \"exit 0\"')"` (found [here](https://unix.stackexchange.com/questions/29777/how-to-average-time-commands))
