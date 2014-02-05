---
layout: post
status: publish
published: true
title: PHPUnit and Doctrine 2 ORM caching issues
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 60
wordpress_url: http://jamesmcfadden.co.uk/?p=60
date: 2012-08-07 15:36:02.000000000 +01:00
categories:
- PHP
- Doctrine
tags: []
---
I've recently begun setting up test suite for a project I'm working on with PHPUnit. The project is built in Zend Framework and makes use of the Doctrine ORM. Doctrine is pretty awesome most of the time, but I've noticed sometimes it tries to do some clever caching when you might not like it.

So I had my basic PHPUnit class which, from within the setUp() function, aimed to establish an in-memory database for the purpose of each individual test. This would reset for each test, leaving me with a nice clean database ready for the next unit:

    public function setUp() 
    {       
        $front = Zend_Controller_Front::getInstance();
        $bootstrap = $front->getParam('bootstrap');
            
        if(!$boostrap) {
            $application    = Zend_Registry::get('application');
            $bootstrap  = $application->getBootstrap();
        }
        $bootstrap->bootstrap('doctrine');
            
        $this->em   = $bootstrap->getResource('doctrine');
        $tool       = new \Doctrine\ORM\Tools\SchemaTool($this->em);
        $classes    = $this->em->getMetaDataFactory()->getAllMetaData();
            
        $tool->dropSchema($classes);
        $tool->createSchema($classes);
            
        Zend_Registry::set('doctrine', $this->em);
            
        parent::setUp();
    }

However, I encountered some bizarre issues when running the following tests (simplified):

    public function testTestOne() 
    {
        $value = 5;
        $model = new Model();
        $model->setUserId($user->getId());
        $model->setValue($value);

        $this->em->persist($model);
        $this->em->flush();

        // Code here to grab the most recent $model from the database

        $this->assertEquals($value, $dbResult->getValue());
    }

    public function testTestTwo() 
    {
        $value = 3;
        $model = new Model();
        $model->setUserId($user->getId());
        $model->setValue($value);
        
        $this->em->persist($model);
        $this->em->flush();

        // Code here to grab the most recent $model from the database

        $this->assertEquals($value, $dbResult->getValue());
    }

The second test failed. I firstly checked that the dataset was clean at the start of the second test, which it was. No 'model' records to be found. 

A quick post-flush var_dump showed that $dbResult->getValue() (the only result in the entire dataset) was returning 5, the value expected from the first test. However, $model->getValue() was set to 3, as it should be, meaning that the value was being changed during the flush.

After a fair bit of messing around and rooting through the Doctrine source, I managed to get the thing to work; it's super simple:

    $this->em->clear();

This line forces the Entity Manager to detach all managed entities, meaning that it can't cache a previously used entity. Adding this before you generate the schemas should ensure you get a completely clean database ready for each test:

    $this->em->clear();

    $tool = new \Doctrine\ORM\Tools\SchemaTool($this->em);
    $classes = $this->em->getMetaDataFactory()->getAllMetaData();
    $tool->dropSchema($classes);
    $tool->createSchema($classes);
