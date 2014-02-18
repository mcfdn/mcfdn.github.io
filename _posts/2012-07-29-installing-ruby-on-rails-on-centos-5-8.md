---
layout: post
status: publish
published: true
title: Installing Ruby on Rails on CentOS 5.8
description: CentOS generally enjoys having out-dated software, such as Ruby 1.8.5. I wanted to get rails running on my VPS to have a play around, which required me to update rubygems and ruby
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 26
wordpress_url: http://jamesmcfadden.co.uk/?p=26
date: 2012-07-29 20:51:18.000000000 +01:00
categories:
- Ruby on Rails
tags: []
---
CentOS generally enjoys having out-dated software, such as Ruby 1.8.5. I wanted to get rails running on my VPS to have a play around, which required me to update rubygems, in turn requiring an update of Ruby itself. Turns out this is more of a pain than one would usually anticipate due to a lack of updated Ruby packages that yum offers.

With the aid of a helpful post at [http://www.rabblemedia.net/installing-rvm-ruby-rails-and-passenger-centos-6](http://www.rabblemedia.net/installing-rvm-ruby-rails-and-passenger-centos-6), I managed to get it working...

First download the Ruby Version Manager (RVM) installer as follows:

    sudo bash -s stable < <(curl -s https://raw.github.com/wayneeseguin/rvm/master/binscripts/rvm-installer )

RVM requires us to add relevant users to it's group. Open the file like so:

    sudo vim /etc/group

Edit the 'rvm' group (likely the last entry) to look like:

    rvm:x:503:your_username

Run the install:

    cd /usr/local/rvm/bin
    sudo ./rvm-installer
    ...
    "Upgrade of RVM in /usr/local/rvm/ is complete."

I needed to reload the RVM to continue, this might not be required:

    rvm reload

Now we can actually install the Ruby version we want. This will also install rubygems which is always useful...

    rvm install 1.9.3 --with-openssl-dir=/usr

Version check seems good:

    ruby -v
    "ruby 1.9.3p194 (2012-04-20 revision 35410) [x86_64-linux]"

We can force RVM to use 1.9.3 like so:

    rvm use 1.9.3 --default

And now for rails:

    gem install rails -V
    rails -v
    Rails 3.2.7

Spiffing
