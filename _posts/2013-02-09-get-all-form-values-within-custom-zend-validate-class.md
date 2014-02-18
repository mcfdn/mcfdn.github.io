---
layout: post
status: publish
published: true
title: Get all form values within custom zend validate class
description: A quick one I recently learned about, Zend_Validate_Interface::isValid allows an optional second argument in addition to the required value parameter
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 476
wordpress_url: http://jamesmcfadden.co.uk/?p=476
date: 2013-02-09 18:20:30.000000000 +00:00
categories:
- Uncategorized
tags: []
---
A quick one I recently learned about, `Zend_Validate_Interface::isValid` allows an optional second argument in addition to the required `value` parameter:

    public function isValid($value, $context = null)
    {
        Zend_Debug::dump($context);
        die;
    }

When used as a form element validator, `$context` will contain all request data including additional form values. This is useful when your validator depends on values only available in the same request.
