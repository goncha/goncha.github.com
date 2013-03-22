---
layout: post
title: "Install Jekyll on Windows"
tags: ruby jekyll windows
---

My laptop used in my daily work is running Windows XP (it's old, by simple). It's not painful to install [Jekyll](https://github.com/mojombo/jekyll/) on it.

**Step1**: download ruby win32 installer from http://rubyinstaller.org/downloads/ and install it.

**Step2**: download Ruby win32 DevKit from http://rubyinstaller.org/downloads/ too. Unpack it at somewhere.

**Step3**: Ctrl+R, cmd to open a console window, change directory to DevKit's unpack directory, then execute the following two lines (**Keep the console window after this step**):

    > ruby dk.rb init
    > ruby dk.rb install

**Step4**: Use gem to install [Jekyll](https://github.com/mojombo/jekyll/).

    > gem install jekyll -v 0.12.0

Now, you can use [Jekyll](https://github.com/mojombo/jekyll/) to write your site.