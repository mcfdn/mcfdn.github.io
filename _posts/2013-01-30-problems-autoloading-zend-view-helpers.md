---
layout: post
status: publish
published: true
title: Problems autoloading Zend view helpers
description: Case sensitive issues encountered with Zend view helpers on Windows
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 413
wordpress_url: http://jamesmcfadden.co.uk/?p=413
date: 2013-01-30 15:10:58.000000000 +00:00
categories:
- PHP
- Zend Framework
tags: []
---
I mainly develop on Windows, and I have recently received an error a couple of times after deploying my Zend Framework applications to Linux boxes:

    Fatal error: Uncaught exception 'Zend_Loader_PluginLoader_Exception' 
    with message 'Plugin by name 'MyPlugin' was not found in the registry;

This error was odd as I knew the files existed with the correct permissions. I found the issue arises when I've been using a custom View object like so:

    $view = new App_View();
    $view->addHelperPath('/App/View/Helper', 'App_View_Helper');

    $viewRenderer = new Zend_Controller_Action_Helper_ViewRenderer();
    $viewRenderer->setView($view);

Where 'App' is my custom library within /library. The error is down to the preceding slash when adding the view helper path; the above should instead read:

    $view->addHelperPath('App/View/Helper', 'App_View_Helper');

Although I would have thought this would have been automatically trimmed, removing the slash resolves the error.
