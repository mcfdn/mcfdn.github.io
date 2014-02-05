---
layout: post
status: publish
published: true
title: Creating a New Remote Git Repository
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 236
wordpress_url: http://jamesmcfadden.co.uk/?p=236
date: 2012-10-01 19:52:36.000000000 +01:00
categories:
- Git
tags: []
---
When not using GitHub to host your Git repositories, you don't have the luxury of simply cloning once you've created the repository. There are a few extra steps documented here:

### Remote Environment

    mkdir NewRepo.git
    cd NewRepo.git

    # Create an empty repository
    git --bare init

    # Set the repository as shared
    git config core.sharedrepository 1

    # Keep the remote repository in control
    git config receive.denyNonFastforwards true

    sudo chown -R youruser *
    sudo chgrp -R youruser *

### Local Environment

    mkdir NewRepo
    cd NewRepo

    git init
    git remote add origin ssh://youruser@yourserver.com:/repositories/NewRepo.git

    # Give us something to commit
    touch README.md
    
    git add .
    git commit -m 'Initial commit'

    # This will establish the master branch allowing others to clone the repository from now on
    git push origin master

