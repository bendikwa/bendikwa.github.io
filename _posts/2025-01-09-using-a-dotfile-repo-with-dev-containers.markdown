---
layout: single
title:  "Using a dotfile repo with Dev containers"
date:   2025-01-08 11:30:00 +0100
categories: VSCode Devcontainer
---
When working with `Dev containers` in `Visual Studio Code` (VSCode) some things happen behind the scenes to help you. One of the things that happen is that the plugin will copy your `.gitconfig` into the container for you.

This might work perfectly well, but if you, like me use the [1 Password SSH Agent](https://developer.1password.com/docs/ssh/agent/) to sign my commits in `GIT` using my `SSH key`, you can end up with having `Windows` specific config in your `.gitconfig`. In my case it is paths to executables that, naturally, do not exist in the container.

One way of solving this is to configure the `Dev container plugin` to fetch dotfiles from a separate repository when building the container. This is also useful if you have other dotfiles you want to have loaded like aliases etc.

# 1 - Create a repository
First you have to create a repository for the files you want to include. It can have any name you like.

# 2 - Add dotfiles to the repo
Add the files that you want to include in your Dev container in the repo. Im my case, I only needed a modified version of my .gitconfig

# 3 - Add an install script to the repo
The Dev container plugin can execute a script after the repository is cloned to copy the dotfiles to the correct locations.

My install.sh looks like this:

{% highlight shell %}
#!/bin/sh
cp .gitconfig ~/
{% endhighlight %}

Remember to set the executable flag on the script before commiting it. If you are on `Linux/Mac` you do it with `chmod +x install.sh` and if you are on `Windows` you can do it with `git update-index --chmod=+x install.sh` after you have added it to `GIT`
{: .notice}

# 4 - Configure the Dev container plugin
In `VSCode` go to “File -> Preferences -> Settings”. Under “Extensions”, locate “Dev container” and scroll down until you find settings for “Dotfiles”.
- Install command: Here you enter the name of the install script in your dotfiles repo
- Repository: Here you enter the git@ link for your repo
- Target path: The directory that the plugin should clone the repo to.

For me the finished config looks something like this:
<img src="{% link /assets/images/dotfiles/settings.png %}" alt="Image showing the settings section for dotfiles in the configuration of the Dev container plugin">

# 5 - Profit!
From now on all `Dev containers` built by the plugin will have your dotfiles in them from the start.