+++
title = "From GUI to TUI: My Journey with git"
date = 2025-03-29
[taxonomies]
tags=["Programming"]
+++

{% callout(label="Note") %}
For any non-programmers reading this, I am not using the word git in the pejorative sense.
[git](https://git-scm.com/) is a distributed version control system.
Essentially, it is a tool that developers use to store code and coordinate their work.
{% end %}

I don't remember my first encounter with `git`.
It was probably one of my more intermediate classes in college where we had to turn in a final project to GitHub.
I'm sure I just stumbled around with `git` in Powershell or Command Prompt until I happened to find the right incantation.
I got by with knowing the least amount that I could about `git` while still getting my assignments done.

I got my first programming job while I was still in college, and I don't actually remember using `git` there.
I'm sure I did some, I just don't remember it.
Using `git` wasn't significant or meaningful to me, it was just something I did because my industry labeled it a "best practice."

After I was out of college and at my next job, I started to understand `git` better just by working with it more.
At some point after college, I realized that VSCode had a nice `git` integration that made it easier to work with.
But looking back, my workflow was still very primitive.
I would add a few files and modify a couple of other files, then I might look through the unstaged files, write a quick commit message, then I would commit my changes.
I don't think I even understood what staging a change even meant.
It was just some weird quirk of using `git` that I accepted.
I think I even turned on some setting in VSCode so it would auto-stage all my changes when I went to make a commit.

I was content using `git` in its simplest form, but I lacked the necessary understanding to do anything more with `git`.
I struggled my way through merge conflicts without truly understanding why they were happening.
If I messed something up on my branch or local repo, I would have to lookup the right reset command or ask someone else for help.
But things changed when I started using NeoVim as my editor.

Before I used NeoVim, I didn't really understand the terminal.
I understood how to copy and paste commands so that I could set up different frameworks or use Docker, but I never truly appreciated the terminal or the shell for what they were.
But I started to get an appreciation for the terminal when I watched [ThePrimeagen](https://www.youtube.com/@ThePrimeagen) code in his YouTube videos.
He was using tools that I had never heard of, to do things in the terminal that I couldn't even understand.
Even though I couldn't fully understand what he was doing at the time, I recognized the power of the terminal.
I knew it was something that I needed to understand if I wanted to become a better developer.
So, one day I decided I needed to fully switch over if I was ever going to learn.
I quit VSCode cold turkey and started using NeoVim for all my coding.

Getting used to NeoVim honestly didn't even take that long looking back.
But at some point early on in the switch, I realized that quitting VSCode meant I would need to find some other way to use `git`.

At first, I went back to what I learned in college.
Good old `git status`, `git pull`, `git commit -a`, and `git push`.
It was refreshing coming back to this workflow, but I still didn't understand `git` very well.
Now that I was trying to understand the terminal better and become a better developer, I knew I had to take `git` more seriously.
So I found that [Pro Git](https://git-scm.com/book/en/v2) is available to read for free online.
I read through that whole book on my phone while we were on vacation.
That book gave me a whole new appreciation for how `git` works.
All of the sudden, I was able to place those basic commands I was using within the wider context of how `git` works.
I understood that branches are really just pointers to a commit hash.
Now I understood why merge conflicts happened, and I even learned a few basic approaches to fixing them (or avoiding them).
I even learned how the `.git/` folder works and how `git` organizes the commit data.
I learned the power of [content-addressable stores](https://en.wikipedia.org/wiki/Content-addressable_storage).
Taking the time to learn `git` better was an important moment for me as a developer because it taught me the value of digging deeper into a topic.
I could have kept on programming and using `git` without truly understanding it.
No one asked me or expected me to learn more about `git`.
Instead, it was something that I expected of myself.

{% callout(label="Musing") %}
It's been a little while now since I went through the [git book](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell).
I probably need to go through it again.
I'm sure I would pick up a whole lot more now that I have some extra experience under my belt.
{% end %}

After I understood `git` better, I kept using it in the command line for a while.
But eventually, one of the NeoVim YouTubers I like (either [ThePrimeagen](https://www.youtube.com/@ThePrimeagen) or [teej_dv](https://www.youtube.com/@teej_dv)) mentioned how great [vim-fugitive](https://github.com/tpope/vim-fugitive) is.
`vim-fugitive` is a NeoVim plugin that allows you to intuitively use `git` from within NeoVim.
The plugin is almost like an alias for the `git` command that is built in to NeoVim.
It allows you to run `:G ...` within vim, which will then behave as if you ran `git ...` in another terminal.
But what makes the plugin awesome is the extra conveniences it provides.
You can run just `:G` to get a nice status buffer with a bunch of keymappings for quickly performing a whole bunch of different `git` operations.
Running `:G blame` in a file will give you a nice extra buffer on the left side, and you can even press `<Enter>` on any of the blame lines to navigate through to that commit's details.
Fugitive is a very nice `git` wrapper for NeoVim and I would highly recommend it if you are looking for something like that.

But my story with `git` is not done yet, because one day, I stumbled upon this video: [Lazygit creator interviews DHH (Rails creator)](https://youtu.be/_94Ja45AVzU?si=sMLiWZZql24lPheW).
YouTube recommended the video to me because I was on a Ruby on Rails kick and I was watching a lot of DHH's videos.
Listening to the interview made me very curious about this `lazygit` tool, so I went and found this video: [15 Lazygit Features In Under 15 Minutes](https://youtu.be/CPLdltN7wgE?si=ilHPf4kuYXpFXEZV).
Seeing how fast Jesse was able to do some of these `git` operations that I wouldn't even know how to do off the top of my head was impressive.
It was like when I first watched `ThePrimeagen` using NeoVim or the terminal.
I saw just how good someone else's workflow was and I wanted it for myself.

So I installed `lazygit` and started using it instead of `vim-fugitive`.
It took me a couple of days before I was comfortable with it, but now it feels great and I love using it.
I especially love how `lazygit` lays out so much information about your repo all at once.
It has helped me a lot with visualizing my commit history and current working changes.
It has changed the way I think about and use `git`.
Recently, I started using it to only stage selected portions or specific lines within a file.
I always knew that was possible, but `lazygit` makes it so easy to do that I finally tried it out and realized how useful it is.
So, now I can work on multiple changes at once, like adding a blog post to this blog while also writing some CSS tweaks.
Then when I'm ready to commit. I will only stage and commit the files or lines that are relevant to my CSS tweaks.
Then I can stage the new blog post as a separate commit.
This may sound like a small or unimportant feature, but it has been great for keeping changes separate.
I haven't needed to roll back a botched deployment before, but I've heard that keeping commits focused on a single feature/change makes it way easier to roll back bad changes.

My main takeaway from my journey with `git` is the importance of figuring things out for yourself and pushing yourself.
It is great when other people give you advice or teach you things, but ultimately, you are in charge of your learning and development.
You are responsible for your growth as a software engineer.
I am not comfortable remaining at my current skill level.
I always want to be learning something new and pushing myself.
I want to be an expert at what I do.
Because being an expert is more fun.
Thanks for reading!
