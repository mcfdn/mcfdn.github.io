---
layout: post
status: publish
published: true
title: Multiple instances of Zend_Navigation per Zend_View
description: I came across an interesting issue recently when attempting to register multiple Zend_Navigation instances to the same view.
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 203
wordpress_url: http://jamesmcfadden.co.uk/?p=203
date: 2012-09-24 23:11:43.000000000 +01:00
categories:
- PHP
- Zend Framework
tags: []
---
I came across an interesting issue recently when attempting to register multiple Zend_Navigation instances to the same view.

I wanted to have a global navigation container and a footer navigation container within the same request. Naturally, I proceeded as followed (simplified):

    $config1 = new Zend_Config_Xml(CONFIG_PATH . "/navigation/global.xml", "nav");
    $config2 = new Zend_Config_Xml(CONFIG_PATH . "/navigation/footer.xml", "nav");

    $nav1 = new Zend_Navigation();
    $nav1->addPages($config1);

    $nav2 = new Zend_Navigation();
    $nav2->addPages($config2);

    echo $this->view->navigation($nav1);
    echo $this->view->navigation($nav2);

However, I received two navigation outputs, both referencing the first configuration file.

    var_dump($nav2->getPages());

This showed that the pages were being pulled in from the config correctly. So it seems that I can't register two navigation helpers in the same view instance?

    $view = new Zend_View();
    echo $this->view->navigation($nav2);

This confirmed that this was indeed the case, with my output exactly as I wanted.. Hardly a good work-around, however.

The issue is that when we set the Zend_Navigation_Container in `Zend_View_Navigation_Abstract::__construct()`, it establishes the default navigation container in `Zend_View_Helper_Navigation_HelperAbstract`. When it comes to rendering a new navigation instance, the default navigation helper's (`Zend_Navigation_Menu`) `render` function is passed a `null` value. Despite the second instance of `Zend_Navigation` setting the new container as default, the original container is pulled out instead.

My favourite solution for this? Simply bypass this functionality by forcing Zend to render each container exclusively:

    echo $this->view->navigation()->render($nav1);
    echo $this->view->navigation()->render($nav2);

A quick poke around on the web led me to [http://framework.zend.com/issues/browse/ZF-6865](http://framework.zend.com/issues/browse/ZF-6865). It seems this has been a problem for a while and "won't fix".

If editing framework code at was at all acceptable, maybe my solution would have been something like this (see Zend_View_Helper_Navigation::render()):

    public function render(Zend_Navigation_Container $container = null)
    {
    	if(null === $container) {
    		$container = $this->getContainer();
    	}
    	$helper = $this->findHelper($this->getDefaultProxy());
        
    	return $helper->render($container);
    }

Truth is, I'm sure this wasn't included for a very good reason...
