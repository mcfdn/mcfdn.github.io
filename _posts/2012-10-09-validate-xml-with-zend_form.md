---
layout: post
status: publish
published: true
title: Validate XML with Zend_Form
description: I needed a simple way to validate user-inputted XML with Zend_Form, so decided to write a simple
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 317
wordpress_url: http://jamesmcfadden.co.uk/?p=317
date: 2012-10-09 13:21:30.000000000 +01:00
categories:
- PHP
- Zend Framework
tags: []
---

I needed a simple way to validate user-inputted XML with Zend_Form, so decided to write a simple validator:

    class App_Validate_Xml extends Zend_Validate_Abstract
    {
        const ERROR_MESSAGE = 'error';
        
        public $xmlError;
        
        protected $_messageVariables = array(
            'xmlError' => 'xmlError'
        );
        
        protected $_messageTemplates = array(
            self::ERROR_MESSAGE => '%xmlError%'
        );
        
        public function isValid($value)
        {
            libxml_use_internal_errors(true);
            
            $dom = new DOMDocument('1.0', 'utf-8');
            $dom->loadXML($value);
            
            $errors = libxml_get_errors();
            
            if(count($errors) > 0) {
                foreach($errors as $error) {
                    $this->xmlError = $error->message;
                    $this->_error(self::ERROR_MESSAGE);
                }
                return false;
            }
            return true;
        }
    }

The validator simply loads the XML from the form element value with [DomDocument](http://php.net/manual/en/class.domdocument.php).

By forcing [libxml](http://php.net/manual/en/book.libxml.php) to allow us to retrieve error information internally, we can check for any errors ourself. It's then just a case of mapping any error messages to our message template ready for Zend_Validate_Abstract to parse them.
