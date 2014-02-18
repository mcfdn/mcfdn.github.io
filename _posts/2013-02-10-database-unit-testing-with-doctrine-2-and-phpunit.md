---
layout: post
status: publish
published: true
title: Database Unit Testing with Doctrine 2 and PHPUnit
description: Settings up Database unit or integration testing with Doctrine 2 ORM and PHPUnit
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 397
wordpress_url: http://jamesmcfadden.co.uk/?p=397
date: 2013-02-10 20:57:16.000000000 +00:00
categories:
- PHP
- Doctrine
tags: []
---
Database unit testing can sometimes be a bit tricky to get working well, especially when using Doctrine. In an [earlier post](/phpunit-and-doctrine-2-orm-caching-issues) I showed how I overcame caching issues I had experienced when using Doctrine with PHPUnit. This post builds upon some of that post, while making use `PHPUnit_Extensions_Database_TestCase`, and assumes you are using the pdo_sqlite in-memory driver.

If you haven't already, you should create a Doctrine connection config specific for unit testing. Something like:

    resources.doctrine.user     = ""
    resources.doctrine.password = ""
    resources.doctrine.host     = ""
    resources.doctrine.dbname   = ""
    resources.doctrine.driver   = "pdo_sqlite"
    resources.doctrine.memory   = "true"

This will, of course, allow for use of your existing Doctrine implementation for testing purposes.

Extend `PHPUnit_Extensions_Database_TestCase` and implement the getConnection() method as required:

    public function getConnection()
    {
        // Get an instance of your entity manager
        $entityManager = $this->getEntityManager();

        // Retrieve PDO instance
        $pdo = $entityManager->getConnection()->getWrappedConnection();

        // Clear Doctrine to be safe
        $entityManager->clear();

        // Schema Tool to process our entities
        $tool = new \Doctrine\ORM\Tools\SchemaTool($entityManager);
        $classes = $entityManager->getMetaDataFactory()->getAllMetaData();

        // Drop all classes and re-build them for each test case
        $tool->dropSchema($classes);
        $tool->createSchema($classes);

        // Pass to PHPUnit
        return $this->createDefaultDBConnection($pdo, 'db_name');
    }

So this method assumes you have already bootstrapped Doctrine elsewhere in your application setup (your application bootstrap, perhaps...). Since we are using an in-memory database, we need to tell Doctrine to process all of our entities. This, of course, will happen for each test case which is exactly what we want; no dependencies and a clean slate to test on.

Finally, we retrieve the pre-configured instance of PDO from Doctrine and pass it on to PHPUnit.

What we are left with is a solution that makes use of the existing Doctrine configuration, reducing repetition while allowing us to rebuild the schema for each test.
