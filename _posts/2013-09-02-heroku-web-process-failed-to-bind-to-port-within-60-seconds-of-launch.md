---
layout: post
status: publish
published: true
title: ! 'Heroku: Web process failed to bind to $PORT within 60 seconds of launch'
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 524
wordpress_url: http://jamesmcfadden.co.uk/?p=524
date: 2013-09-02 16:53:56.000000000 +01:00
categories:
- heroku
- node.js
tags: []
---
While deploying a node.js application to Heroku recently, I was receiving the following error:

    Error R10 (Boot timeout) -> Web process failed to bind to $PORT within 60 seconds of launch

A common cause for this error is due to the fact that Heroku dynamically assigns a port to your application process, something that doesn't seem to be mentioned in the [documentation](https://devcenter.heroku.com/articles/nodejs). If you have forced a port number in your node server, Heroku will fail to bind to it.

The following line will ensure your application makes use of the port assigned to the user environment by Heroku:

    var port = process.env.PORT || 5000;