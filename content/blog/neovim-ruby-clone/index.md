+++
title = "Making a NeoVim Clone with Ruby"
date = 2025-05-06
[taxonomies]
tags=["Programming", "Ruby"]
+++

I've been learning Ruby recently.
I've really been enjoying it and I've been reading through the Ruby docs quite a bit.
When I found this section on [`io/console`](https://docs.ruby-lang.org/en/3.4/IO.html#class-IO-label-Extension+io-2Fconsole), I thought it sounded like fun.
Since I have grown quite attached to using NeoVim as my editor, I thought it would be fun to make a NeoVim clone.
For now, I'm calling it RVim, which is short for Ruby Vim.
A high-quality implementation of a text editor would require <i>tons</i> of code to handle all the little edge cases around text manipulation, buffer display, and cursor handling.
Obviously RVim is a long way off from handling all of that correctly, but I am surprised by how many features I've been able to pack into ~600 lines of code.
I'm attributing that to how expressive Ruby is, and how well suited it is as a scripting language.

One example of how productive Ruby can be comes from how I implemented the command mode.
NeoVim has a command mode (`:help Cmdline-mode` in Vim to learn more) which allows users to run a variety of commands.
To implement something exactly equivalent to Vim's command mode would essentially require a custom language parser and interpreter.
As much as I enjoy making little languages, I realized I could cheat by just providing a Ruby REPL instead.
To implement a basic version of this REPL, I made a `RVimCommandCenter` class which holds all the helper methods I want to expose to users.
I made a basic eval method on `RVimCommandCenter` which will evaluate input within the context of the class instance by using `binding.eval input`.
Any code evaluated using this method will have immediate access to the methods within `RVimCommandCenter`.
So, to implement the `q` command to quit the editor, I can simply do this:

```ruby
def q
  @state.should_quit = true
end
```

I'm still in the early stages of building out this command system, but it is surprisingly powerful.
For now, it's not useful as a general purpose REPL because it does not display the results of the evaluation to the user, but that can always be fixed later on.
In the demo video below, you can see how I directly access the state variable on the command center class to monkey around with the internal state of the editor.
In addition to providing some convenient alias methods that mirror NeoVim's commands, the command center class also allows users full access to the internal state of the editor.
Giving users that kind of power can be dangerous, but I believe the benefits far outweigh the risks.
By implementing the editor in pure Ruby and exposing the editor state as much as possible, I believe RVim could be extremely introspectable and customizable with little implementation effort on my part.
With the right API design on the command center and state classes, it could even be easier to extend than NeoVim is.
But we are still a ways off from that point.

To provide a better REPL experience while I am still building out the command mode, I also added a `dbg` method to the command center which will place the console back into cooked mode and open [irb](https://github.com/ruby/irb).
This will give users a much better REPL editing experience with autocomplete and history support.
You can also see me using irb to debug the editor state in the below video.
Currently you cannot use irb's full debugging toolkit because the application will go back into raw mode as soon as you step out of the initial breakpoint, but I think that could be fixed without too much effort.

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/hzL5Su0PDo0?si=w15WHxyCsWxge_fd" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

I hope you enjoyed reading about my latest crazy project.
Even if it never takes off or becomes a useful editor, I have already learned a lot about Ruby and I've had a lot of fun.
[Here](https://github.com/EliasPrescott/rvim) is the repo for RVim if you want to look at the code or play around with it.
You should be able to run `bundle` or `gem install -g`, and then just `ruby main.rb`.
