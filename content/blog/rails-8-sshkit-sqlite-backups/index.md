+++
title = "Simple SQLite Backups for Rails 8 with SSHKit"
date = 2025-04-28
[taxonomies]
tags=["Programming", "Rails"]
+++

This won't be a long post, I just wanted to record something I struggled with for a little while.
I recently made a quick toy app using Rails 8 as a learning experience ([emoji-polls.australorp.dev](https://emoji-polls.australorp.dev)).
I don't actually need database backups since this is only a toy app, but I thought it sounded like a fun project that wouldn't take too long.
It ended up taking longer than I thought because I had to play around with the user and group permissions on my VM a little bit, and directly accessing a named Docker volume from the host side turned out to be more difficult than I thought.
Eventually, I realized that I could just use `docker volume inspect $VOLUME_NAME` and then parse out the mount point property.
I tried to mess with the folder permissions so my admin user could access the docker volumes without using `sudo`, but I eventually gave up and started using `sudo`, and then everything worked perfectly.
Sometimes, [worse really is better](https://en.wikipedia.org/wiki/Worse_is_better).

This project was also a fun excuse to play around with [SSHKit](https://github.com/capistrano/sshkit), which is a Ruby library for running commands on remote servers.
My use case was really simple since I only have one host and I don't have any complex logic.
My favorite feature is the ability to [transfer files](https://github.com/capistrano/sshkit?tab=readme-ov-file#transferring-files).
Before settling on using SSHKit, I was looking into various other methods for automating this process using bash scripting, and none of them looked fun.
But SSHKit makes transferring a file back to the host side super simple.
This was my first experience replacing some of my normal bash scripting with Ruby, but it was a really positive one.

```ruby
#!/usr/bin/env ruby
require "sshkit"
require "sshkit/dsl"
require "date"
require "json"
include SSHKit::DSL

SSHKit::Backend::Netssh.configure do |ssh|
  ssh.ssh_options = {
    user: 'admin',
    keys: ['.ssh/kamal'],
    forward_agent: false,
    auth_methods: ['publickey'],
  }
end

environment = "production"

on 'emoji-polls.australorp.dev' do
  file_name = "#{DateTime.now.strftime("%Y-%m-%d")}-#{environment}.db"
  temp_file_path = "/tmp/#{file_name}"
  local_image_path = "./images/#{file_name}"
  info "Backing up image to #{temp_file_path} on #{host}"
  volumes_output = capture :docker, "volume", "inspect", "emoji_polls_storage"
  volumes_info = JSON.parse volumes_output
  volume_path = volumes_info[0]["Mountpoint"]
  execute :sudo, :sqlite3, "#{volume_path}/#{environment}.sqlite3", "'.backup #{temp_file_path}'"
  download! temp_file_path, local_image_path
  info "Deleting image on #{host}"
  execute :sudo, :rm, temp_file_path
end
```
