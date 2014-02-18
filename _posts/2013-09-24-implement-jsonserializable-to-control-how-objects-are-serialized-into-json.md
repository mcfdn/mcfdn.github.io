---
layout: post
status: publish
published: true
title: Implement JsonSerializable to control how objects are serialised into JSON
description: I assumed that json_encode would make use of PHP's __toString() magic method, giving me the ability to manage how my object is processed. Turns out it doesn't
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 547
wordpress_url: http://jamesmcfadden.co.uk/?p=547
date: 2013-09-24 20:49:21.000000000 +01:00
categories:
- PHP
tags: []
---

I assumed that json_encode would make use of PHP's `__toString()` magic method, giving me the ability to manage how my object is processed. Turns out it doesn't. However, if you're using PHP 5.4 or later, you can make use of the [JsonSerializable](http://php.net/manual/en/class.jsonserializable.php) interface which allows specification of what object data should be serialised:

    /**
     * @author James McFadden <james@jamesmcfadden.co.uk>
     */
    class Nova_Dto_Vacancy implements JsonSerializable
    {
        protected $_vacancy;
        
        public function __construct(Model_Vacancy $vacancy)
        {
            $this->_vacancy = $vacancy;
        }
        
        public function getData()
        {
            return array(
                'Name' => $this->_vacancy->getName()
            );
        }
        
        public function jsonSerialize()
        {
            return $this->getData();
        }
    }
