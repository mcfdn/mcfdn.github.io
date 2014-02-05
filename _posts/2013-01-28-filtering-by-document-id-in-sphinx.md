---
layout: post
status: publish
published: true
title: Filtering by Document ID in Sphinx
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 417
wordpress_url: http://jamesmcfadden.co.uk/?p=417
date: 2013-01-28 21:01:55.000000000 +00:00
categories:
- PHP
- Sphinx
tags: []
---
I wanted to exclude a certain document ID from within Sphinx:

    sql_query = \
        SELECT u.id as u_id from users u
    sql_attr_uint = u_id

and then in PHP:

    $sphinxClient->addFilter('u_id', $userId, true);

As you are unable to filter directly on IDs, this approach yielded no results.

What you need to do is create a copy of the column and alias it for your attribute:

    sql_query = \
        SELECT u.id, u.id as u_id from users u
    sql_attr_uint = u_id

Now using the previous PHP to add the filter should return the results as expected.
