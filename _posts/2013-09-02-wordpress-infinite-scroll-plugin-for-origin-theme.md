---
layout: post
status: publish
published: true
title: Wordpress Infinite Scroll plugin for Origin theme
description: I've just added infinite scrolling to the blog, with the help of the Infinite-Scroll plugin.
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 539
wordpress_url: http://jamesmcfadden.co.uk/?p=539
date: 2013-09-02 17:45:27.000000000 +01:00
categories:
- Uncategorized
tags: []
---
I've just added infinite scrolling to the blog, with the help of the [Infinite-Scroll](http://wordpress.org/plugins/infinite-scroll) plugin. Since I couldn't find any settings around for my current Wordpress theme ([Origin](http://wordpress.org/themes/origin)), I thought I'd post them here in the hope that they might help someone:

| Option                | Value                 |
| ----------------------|-----------------------|
| Content Selector      | #content .hfeed       |
| Navigation Selector   | .pagination           |
| Next Selector         | .pagination a.next    |
| Item Selector         | .hentry.post          |

<small>Note these are confirmed working with Origin Theme Version 0.5.6, untested elsewhere!</small>
