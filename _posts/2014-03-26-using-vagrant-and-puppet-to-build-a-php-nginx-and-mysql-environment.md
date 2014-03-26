---
layout: post
title: Using Vagrant and Puppet to build a PHP, Nginx and MySQL environment
description: I wanted to set up up a local vagrant instance running PHP, Nginx and MySQL that could act as a base environment for other projects I'm working on. These are the steps I took from start to finish.
---

I wanted to set up a local Vagrant instance running PHP, Nginx and MySQL that could act as a base environment for other projects I'm working on. These are the steps I took from start to finish. The environment is hosted on GitHub [here](https://github.com/jamesmcfadden/vagrant-puppet). 

Originally I intended this to be a step by step "how to", but it has since turned into more of an introduction into Vagrant and Puppet as well. You can [jump straight to the PHP, Nginx and MySQL configuration](#php-nginx-mysql-configuration) if you wish.

### What is Vagrant?

For those who don't know, [Vagrant](http://www.vagrantup.com) is a great tool that helps establish a consistent environment to develop and deploy code within. The [Vagrant documentation](https://docs.vagrantup.com/v2/why-vagrant) puts this perfectly:

>Vagrant provides easy to configure, reproducible, and portable work environments built on top of industry-standard technology and controlled by a single consistent workflow to help maximize the productivity and flexibility of you and your team.

### Install VirtualBox

At the core of Vagrant is a virtual machine. Vagrant supports [several VM providers](https://docs.vagrantup.com/v2/providers), but we'll be using VirtualBox. Download and install VirtualBox from the [official website](https://www.virtualbox.org/wiki/Downloads).

### Install Vagrant

Download and install Vagrant from the [official website](http://www.vagrantup.com/downloads.html).

### Add your base box

A base box is the "default" image that is used to build your environment. It avoids the need to download an image each time a box is provisioned, leading to a quicker startup time.

First, let's create a directory for our Vagrant environment:

    $ mkdir vagrant ; cd vagrant

It is possible to create your own base box, and there are many [community-managed boxes](http://www.vagrantbox.es) available, but for now we are going to use the Vagrant standard `hashicorp/precise32`:

    $ vagrant box add hashicorp/precise32

This will download and set a 32-bit Ubuntu image to be our base box.

### Configuring Vagrant

The next step is to configure our vagrant instance. We do this in the `VagrantFile`.

Running `vagrant init` will create a `VagrantFile` for us with some default configuration values inside. However, for now we want to configure our environment ourself:

    vim VagrantFile

Add the following:

    Vagrant.configure("2") do |config|
      config.vm.box = "hashicorp/precise32"
      config.vm.network :forwarded_port, host: 5000, guest: 80
    end

This is the most basic `VagrantFile`, and isn't far off what Vagrant would have given us had we run `vagrant init`.

We can see that we're setting the box to use `hashicorp/precise32`, which we established as our base box earlier on.
We're also thinking ahead here and forwarding all network activity on port 5000 to port 80 within our Vagrant guest machine.

### Vagrant up

At this point, our environment should be ready for (basic) use. To create a Vagrant instance from nothing run: 

    vagrant up

This will provision the box based on the settings in `VagrantFile`. Once complete we can ssh into our box: 

    vagrant ssh

Awesome! However, all we have is an empty, Ubuntu-default box. No installed packages..

### Using Puppet to automate environment configuration

While we could just provision a shell script in `VagrantFile` to install our software packages for us, a much better solution would be to use [Puppet](http://puppetlabs.com). Puppet will allow us to automate our box deployment even further by installing all the software we want and configuring it as we need.

We'll start by creating some directories for Puppet to work from:

    # Exit the guest box if you are logged in
    $ exit

    # Inside the vagrant directory on the host system
    $ mkdir -p puppet/{manifests,modules}

    # We'll also make a directory for our nginx-served files
    $ mkdir app

Now we'll create our base Puppet 'manifest':

    $ vim puppet/manifests/init.pp

Inside we'll put:

    exec { 'apt-get update':
      path => '/usr/bin',
    }

    package { 'vim':
      ensure => present,
    }

    file { '/var/www/':
      ensure => 'directory',
    }

Here we are instructing Puppet to:

- Run apt-get update;
- Ensure the Vim package is installed and present;
- Ensure the `/var/www` directory is present.

We now need to instruct Vagrant to provision Puppet at install. We do this by adding the following to our `VagrantFile`:

    config.vm.provision :puppet do |puppet|
        puppet.manifests_path = 'puppet/manifests'
        puppet.module_path = 'puppet/modules'
        puppet.manifest_file = 'init.pp'
      end

Our `VagrantFile` now looks like:

    Vagrant.configure("2") do |config|
      config.vm.box = "hashicorp/precise32"
      config.vm.network :forwarded_port, host: 5000, guest: 80

      config.vm.provision :puppet do |puppet|
        puppet.manifests_path = 'puppet/manifests'
        puppet.module_path = 'puppet/modules'
        puppet.manifest_file = 'init.pp'
      end
    end

Now we can reload the box and force Vagrant to run any provisioners such as Puppet:

    vagrant reload --provision

Some additional information will be output this time:

    notice: /Stage[main]//Exec[apt-get update]/returns: executed successfully
    notice: /Stage[main]//File[/var/www/]/ensure: created
    notice: /Stage[main]//Package[vim]/ensure: ensure changed 'purged' to 'present'
    notice: Finished catalog run in 15.62 seconds

If we now `vagrant ssh` into our box, we should see that `vim` has been installed, and our web directory exists at `/var/www`. Automation is cool!

<a name="php-nginx-mysql-configuration">
</a>
### Installing Nginx, MySQL and Puppet

So far, we are able to provision a Vagrant box and automate the installation of packages. We could continue to add configuration within `puppet/manifests/init.pp`, but this is going to become hard to maintain.

A better way is to separate our dependencies out into logically separated manifests.

We'll start by creating our directory structure. Puppet (quite rightly) expects a consistent directory structure for its modules:

    $ cd puppet/modules
    $ mkdir -p nginx/{files,manifests}
    $ mkdir -p php/{files,manifests}
    $ mkdir -p mysql/{files,manifests}

Next we need to add the following line to the end of our `puppet/manifests/init.pp`:

    include nginx, php, mysql

This will ensure our `nginx`, `php` and `mysql` manifests are included during the Vagrant provision.

Now let's take a look at a basic nginx configuration:

    # In puppet/modules
    vim nginx/manifests/init.pp

We'll define our nginx class like so:

    # vagrant/puppet/modules/nginx/manifests/init.pp
    class nginx {

      # Symlink /var/www/app on our guest with 
      # host /path/to/vagrant/app on our system
      file { '/var/www/app':
        ensure  => 'link',
        target  => '/vagrant/app',
      }

      # Install the nginx package. This relies on apt-get update
      package { 'nginx':
        ensure => 'present',
        require => Exec['apt-get update'],
      }

      # Make sure that the nginx service is running
      service { 'nginx':
        ensure => running,
        require => Package['nginx'],
      }

      # Add a vhost template
      file { 'vagrant-nginx':
        path => '/etc/nginx/sites-available/127.0.0.1',
        ensure => file,
        require => Package['nginx'],
          source => 'puppet:///modules/nginx/127.0.0.1',
      }

      # Disable the default nginx vhost
      file { 'default-nginx-disable':
        path => '/etc/nginx/sites-enabled/default',
        ensure => absent,
        require => Package['nginx'],
      }

      # Symlink our vhost in sites-enabled to enable it
      file { 'vagrant-nginx-enable':
        path => '/etc/nginx/sites-enabled/127.0.0.1',
        target => '/etc/nginx/sites-available/127.0.0.1',
        ensure => link,
        notify => Service['nginx'],
        require => [
          File['vagrant-nginx'],
          File['default-nginx-disable'],
        ],
      }
    }

The comments should be self-explanatory here. We basically install and run Nginx and serve the `/var/www/app` directory which is accessible through the host system (read more about [synced folders](http://docs.vagrantup.com/v2/synced-folders/)). Finally we create and enable a vhost based on a template we will create next. Note that `puppet:///modules/nginx/foo` will reference `puppet/modules/nginx/files/foo` on the host.

Our Nginx vhost is simple:

    # vagrant/puppet/modules/nginx/files/127.0.0.1
    server {
      listen 80;
      server_name _;
      root /var/www/app;
      index index.php;

      location / {
        try_files $uri /index.php;
      }

      location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        include fastcgi_params;
      }
    }

So that's Nginx set up. Let's move onto PHP:

    # vagrant/puppet/modules/php/manifests/init.pp
    class php {

      # Install the php5-fpm and php5-cli packages
      package { ['php5-fpm',
                 'php5-cli']:
        ensure => present,
        require => Exec['apt-get update'],
      }

      # Make sure php5-fpm is running
      service { 'php5-fpm':
        ensure => running,
        require => Package['php5-fpm'],
      }
    }

Another pretty obvious class. Sets up PHP5 ([FPM](http://php-fpm.org/)) and PHP5 CLI.

Finally, we'll configure how Puppet handles MySQL:

    # vagrant/puppet/modules/mysql/manifests/init.pp
    class mysql {

      # Install mysql
      package { ['mysql-server']:
        ensure => present,
        require => Exec['apt-get update'],
      }

      # Run mysql
      service { 'mysql':
        ensure  => running,
        require => Package['mysql-server'],
      }

      # Use a custom mysql configuration file
      file { '/etc/mysql/my.cnf':
        source  => 'puppet:///modules/mysql/my.cnf',
        require => Package['mysql-server'],
        notify  => Service['mysql'],
      }

      # We set the root password here
      exec { 'set-mysql-password':
        unless  => 'mysqladmin -uroot -proot status',
        command => "mysqladmin -uroot password a9120ed2b58af37862a83f5b9f850819ed08b2a9",
        path    => ['/bin', '/usr/bin'],
        require => Service['mysql'];
      }
    }

This class follows the same format as the Nginx and PHP manifests. The only notable difference is that we manually set the root user password. This is optional, but illustrates how simple MySQL operations may be performed. For anything more than this I would recommend looking into the [MySQL Puppet module](https://forge.puppetlabs.com/puppetlabs/mysql).

I'm opting to leave out the `my.cnf` file to prevent what is an already long post becoming even longer. It's nothing special. It's just a standard [my.cnf](http://www.fromdual.com/mysql-configuration-file-sample) file. You can see it on the [GitHub repository](https://github.com/jamesmcfadden/vagrant-puppet). This should be placed in `vagrant/puppet/modules/mysql/files/my.cnf`.

At this point, you should be able to reload your Vagrant instance using the command we used before:

    $ vagrant reload --provision

Hopefully, everything should work out as expected and you should have Nginx, PHP and MySQL all playing together nicely.

From here you can keep your application code within `/path/to/vagrant/app`, and because we forwarded incoming Vagrant requests on port 5000 to port 80 on the guest machine, it means we can hit the `app` folder through a standard web server request. Therefore, a typical workflow might look like:

    $ git clone git@github.com:/foobar/my_dev_environment
    $ cd my_dev_environment
    $ git clone git@github.com:/foobar/my_application app

    $ vagrant up

For now though, let's finish up by checking our environment is working as expected with a simple PHP script:

    $ echo "<?php phpinfo();" > app/index.php

Direct your browser to `http://127.0.0.1:5000` and you should see the standard PHP info output.

I think that's it. We now have a consistent, deployable Vagrant environment that we can use as a base for greater things! Any feedback in the comments below is appeciated, and feel free to submit any fixes to the repository [here](https://github.com/jamesmcfadden/vagrant-puppet). Thanks for reading!
