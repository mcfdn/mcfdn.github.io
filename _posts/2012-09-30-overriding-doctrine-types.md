---
layout: post
status: publish
published: true
title: Overriding Doctrine Types
description: Sometimes you want to override Doctrine types. For instance, if you have a child project that must adhere to a parent project's Entity Annotations, but you want to use a different class for the Doctrine return value (than the one specified in the parent project), overriding the existing Doctrine type can help.
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 225
wordpress_url: http://jamesmcfadden.co.uk/?p=225
date: 2012-09-30 18:58:47.000000000 +01:00
categories:
- PHP
- Doctrine
tags: []
---
Sometimes you want to override Doctrine types. For instance, if you have a child project that must adhere to a parent project's Entity Annotations, but you want to use a different class for the Doctrine return value (than the one specified in the parent project), overriding the existing Doctrine type can help.

If you simply try to register a new type when a type of the same name has already been registered, you'll get an exception along the lines of:

    Type mydoctrinetype already exists.

A method that some people might not know about (at least I didn't) is `DBAL\Types\Type::overrideType()`. This pretty much solves the problem; although it doesn't assume you wish to register the new type even if a previous one wasn't already added:

    Type to be overwritten mydoctrinetype does not exist

So a little more is required:

    \Doctrine\DBAL\Types\Type::getTypesMap();

This will get you an array of key value pairs of the currently registered type names and their namespaces, and so you can do a quick check to see which function you should be calling:

    $type = "mydoctrinetype";
    $typeNamespace = "my\doctrine\type";
    $typeClass = "MyDoctrineType";
    $registeredTypes = \Doctrine\DBAL\Types\Type::getTypesMap();
    
    if(array_key_exists($type, $registeredTypes)) {
        Doctrine\DBAL\Types\Type::overrideType($type, $typeNamespace);
    } else {
        Doctrine\DBAL\Types\Type::addType($type, $typeNamespace);
    }
    $entityManager->getConnection()->getDatabasePlatform()->registerDoctrineTypeMapping($typeClass, $type);

Providing your Type class is setup correctly, you should see Doctrine correctly uses your new Type in place of it's parent.
