---
layout: post
status: publish
published: true
title: Truncating All Tables in Doctrine
description: A quick script I use to truncate all entities registered in Doctrine
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 273
wordpress_url: http://jamesmcfadden.co.uk/?p=273
date: 2012-10-03 10:41:20.000000000 +01:00
categories:
- PHP
- Doctrine
tags: []
---
A quick script I use to truncate all entities registered in Doctrine:

    $connection = $entityManager->getConnection();
    $schemaManager = $connection->getSchemaManager();
    $tables = $schemaManager->listTables();
    $query = '';

    foreach($tables as $table) {
        $name = $table->getName();
        $query .= 'TRUNCATE ' . $name . ';';
    }
    $connection->executeQuery($query, array(), array());