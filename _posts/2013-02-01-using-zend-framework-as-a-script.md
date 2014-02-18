---
layout: post
status: publish
published: true
title: Using Zend Framework as a script
description: Running Zend Framework as a script can obviously be useful in many circumstances. These are the steps I take to get up and running on the command line.
author: James
author_login: James
author_email: james@jamesmcfadden.co.uk
wordpress_id: 453
wordpress_url: http://jamesmcfadden.co.uk/?p=453
date: 2013-02-01 17:53:19.000000000 +00:00
categories:
- PHP
- Zend Framework
tags: []
---
Running Zend Framework as a script can obviously be useful in many circumstances. These are the steps I take to get up and running on the command line.

### Modify index.php

This is a modified version of the usual /public/index.php. In most cases you could just simply use index.php, but I find a separate script is better as I can specify which Bootstrap resources to load. Additionally I can keep this with the rest of my scripts.

    date_default_timezone_set("Europe/London");

    $acceptedEnvironments = array(
        'development',
        'testing',
        'staging',
        'production'
    );

    $options = getopt("e:");
    $env = $options['e'];

    if(!in_array($env, $acceptedEnvironments)) {
        die('Invalid environment ' . $env . ' given as the first argument');
    }

    // Unset environment key and value as we no longer need it
    unset($_SERVER['argv'][1]);
    unset($_SERVER['argv'][2]);

    define("APPLICATION_ENV", $env);

    // Define path to application directory
    defined('APPLICATION_PATH')
        || define('APPLICATION_PATH',
                  realpath(dirname(__FILE__) . '/../application'));

    // Define application environment
    defined('APPLICATION_ENV')
        || define('APPLICATION_ENV',
                  (getenv('APPLICATION_ENV') ? getenv('APPLICATION_ENV')
                                             : 'production'));

    set_include_path(implode(PATH_SEPARATOR, array(
        dirname(dirname(__FILE__)) . '/library',
        realpath(APPLICATION_PATH),
        get_include_path(),
    )));

    /** Zend_Application */
    require_once 'Zend/Application.php';

    // Create application, bootstrap, and run
    $application = new Zend_Application(
        APPLICATION_ENV,
        APPLICATION_PATH . '/configs/application.ini'
    );
    $application->bootstrap(array('cli', 'autoloader', 'modules'))
                ->run();

### Custom router

The custom router will interpret arguments passed on the command line and route the current request to the specified controller and action. It also keeps any left over parameters for use later on.

    class App_Controller_Router_Cli extends Zend_Controller_Router_Abstract
    {
        public function route(Zend_Controller_Request_Abstract $request)
        {
            $getopt = new Zend_Console_Getopt(array());
            $arguments = $getopt->getRemainingArgs();

            if ($arguments) {
                $controller = array_shift($arguments);
                $action = array_shift($arguments);

                if (!preg_match ('~\W~', $controller)) {
                    $request->setControllerName($controller);
                    $request->setActionName($action);
                    unset($_SERVER['argv'][1]);

                    // Any left over arguments are set as request parameters
                    $request->setParams($arguments);

                    return $request;
                }
                echo "Invalid command given.\n", exit;
            }
            echo "No command given. Expecting a controller name followed by an action name. Example: \n
                [php script.php mail process]\n", exit;
        }

        public function assemble ($userParameters, $name = null, $reset = false, $encode = true) {
            die("Oops! " . __METHOD__ . " might need implementing");
        }
    }

### Bootstrap resource

In your Bootstrap you want to make sure your custom router is only used when on the command line. You'll also want to set a default module. Additionally you may choose to change your default error controller so it's more suited for the command line:

    public function _initCli()
    {
        if(PHP_SAPI == 'cli') {

            $this->bootstrap('FrontController');
            $front = $this->getResource('FrontController');
            $front->setRouter(new App_Controller_Router_Cli());
            $front->setRequest(new Zend_Controller_Request_Simple());
            $front->setDefaultModule('cli');

            $error = new Zend_Controller_Plugin_ErrorHandler(array(
                'module' => 'cli',
                'controller' => 'error',
                'action' => 'error'
            ));
        }
    }

### Usage

You'll then be able to run the following command to execute your chosen action.

    php script.php -e development controllername actionname

This should route to your default script module and the controller/ action as specified.

Don't forget to create your ErrorController to catch any exceptions that might be thrown using [Zend's error handler plugin](http://framework.zend.com/manual/1.12/en/zend.controller.plugins.html#zend.controller.plugins.standard.errorhandler); Zend will otherwise try and redirect to IndexController in your chosen script module.

It's also usually a good idea to create your own abstract script controller that by default turns off view rendering:

    class App_Controller_Cli_Abstract extends Zend_Controller_Action
    {
        public function init()
        {
            parent::init();
            $this->_helper->viewRenderer->setNoRender(TRUE);
        }
    }

Thanks to [this Stack Overflow post](http://stackoverflow.com/a/4706966/1240134) for a nudge in the right direction.
