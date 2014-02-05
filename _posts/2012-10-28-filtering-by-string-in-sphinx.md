---
layout: post
status: publish
published: true
title: Filtering by string in Sphinx
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 348
wordpress_url: http://jamesmcfadden.co.uk/?p=348
date: 2012-10-28 23:56:57.000000000 +00:00
categories:
- PHP
- Sphinx
tags: []
---
Sphinx doesn't support filtering by strings as attributes (yet), so here are two options you can use to get around this limitation...

### Querying by Attribute

This option requires you to define a text field in your Sphinx config and reference said field when querying your index. In your config for the source you are working with:

    SELECT id, user_type \
            FROM users
    sql_field_string = user_type

And in PHP:

    $matches = $this->sphinxClient->Query('@user_type "^admin$"');

### Full-Text Searching a Column

This is my preferred option, as you can still manage string filters with your other integer attributes, and, personally, I think it's easier to set these up together before querying the index(es). In your Sphinx config:

    SELECT id, CRC32(user_type) AS user_type \
            FROM users
    sql_attr_int    = user_type

PHP would look something like this:

    $this->sphinxClient->SetFilter('user_type', array(crc32('admin'));

When using this option, note that there's a [caveat](http://php.net/manual/en/function.crc32.php) in PHP's CRC for integers on 32 bit platforms.
