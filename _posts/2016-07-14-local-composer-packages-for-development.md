---
layout: post
title: Local Composer Packages for Development
description: New PHP packages are usually developed in parallel with larger projects. However, it's often inconvenient to publish an unfinished package in order to require it as a dependency in your project's composer.json.
---

New PHP packages are usually developed in parallel with larger projects. However, it's often inconvenient to publish an unfinished package in order to require it as a dependency in your project's `composer.json`.

Here's how you can require a local composer package in a local project, allowing you to continue development until it's ready for publication. We'll have two separate project folders; **package** and **project**:

### Package composer.json

    {
        "name": "jamesmcfadden/package",
        "description": "My package",
        "license": "MIT",
        "authors": [
            {
                "name": "James McFadden",
                "email": "james@jamesmcfadden.co.uk"
            }
        ],
        "autoload": {
            "psr-4": {
                "App\\": "src"
            }
        },
        "require": {
            "nesbot/carbon": "~1.21"
        }
    }

Here we have a simple `composer.json` which defines our package and a single dependency [`nesbot/carbon`](http://carbon.nesbot.com). We're using the [PSR-4](http://www.php-fig.org/psr/psr-4) autoloader specification, so ensure your project is structured accordingly. Next we'll reference our package as a dependency in our project.

### Project composer.json

    {
        "name": "jamesmcfadden/project",
        "description": "New project",
        "license": "MIT",
        "authors": [
            {
                "name": "James McFadden",
                "email": "james@jamesmcfadden.co.uk"
            }
        ],
        "repositories": [
            {
                "type": "path",
                "url": "/path/to/jamesmcfadden/package"
            }
        ],
        "require": {
            "jamesmcfadden/package": "dev-master"
        }
    }

The key part of our project's `composer.json` is `repositories`; this defines our package as a path, which means we are able to reference a location on our local machine. We include it in our `require` section as we would usually.

We can now run `composer install` in our project root. Our development package will be installed to our `/vendor` directory as a symlink (by default, if possible). This is great, as it means we can make changes to our package without needing to run `composer update` in our project after each change! Note that any changes to your package `composer.json`, however, will require a `composer update` for the changes to take effect, as usual.

When your package is finished and published to a package repository such as [Packagist](https://packagist.org), all thats left to do is switch out/remove your `repositories` section and update your dependencies as appropriate. For example:

    {
        "name": "jamesmcfadden/project",
        "description": "New project",
        "license": "MIT",
        "authors": [
            {
                "name": "James McFadden",
                "email": "james@jamesmcfadden.co.uk"
            }
        ],
        "require": {
            "jamesmcfadden/publishedpackage": "1.0.0"
        }
    }

### Resources

- [Composer Repositories - Path](https://getcomposer.org/doc/05-repositories.md#path)
- [PSR-4: Autoloader](http://www.php-fig.org/psr/psr-4)
