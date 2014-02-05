---
layout: post
status: publish
published: true
title: Preserving numerical array keys while merging
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 498
wordpress_url: http://jamesmcfadden.co.uk/?p=498
date: 2013-05-20 20:42:43.000000000 +01:00
categories:
- PHP
tags: []
---
Here's one I recently learned. Given two arrays:

    $old = array(
        103 => 'Value for 103',
        53 => 'Value for 53'
    );

    $new = array(
        53 => 'New value to replace old 53',
        9376 => 'Value for 9376'
    );

`array_merge` will not preserve numeric keys, meaning that subsequently added items will be added as new values even if you don't intend them to be. To preserve the keys use the array union operator:

    var_dump(array_merge($old, $new));

    array (size=4)
        0 => string 'Value for 103' (length=13)
        1 => string 'Value for 53' (length=12)
        2 => string 'New value to replace old 53' (length=27)
        3 => string 'Value for 9376' (length=14)

    var_dump($new + $old);

    array (size=3)

        53 => string 'New value to replace old 53' (length=27)
        9376 => string 'Value for 9376' (length=14)
        103 => string 'Value for 103' (length=13)

Common knowledge, perhaps, but this was a new one for me :)
