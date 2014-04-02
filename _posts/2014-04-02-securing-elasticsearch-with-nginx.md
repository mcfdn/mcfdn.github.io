---
layout: post
title: Securing Elasticsearch with Nginx
description: Utilising Elasticsearch is a great way to provide full-text search capabilities to any application through a familiar RESTful interface. However, by default it doesn't come with any way to enforce security...
---
Utilising Elasticsearch is a great way to provide full-text search capabilities to any application through a familiar RESTful interface. However, by default it doesn't come with any way to enforce security, so this is left up to the developer.

There are few ways in which this can be done, including through the use of [various](https://github.com/sonian/elasticsearch-jetty) [dedicated](https://github.com/Asquera/elasticsearch-http-basic) [plugins](https://github.com/codelibs/elasticsearch-auth). I, along with others, feel that a more flexible and scalable approach is to use [Nginx](http://wiki.nginx.org/Main) to provide a layer of authentication in front of Elasticsearch. 

Thanks to some [helpful](http://www.ragingcomputer.com/2014/02/securing-elasticsearch-kibana-with-nginx) [posts](http://stackoverflow.com/questions/22785794/restricting-direct-access-to-port-but-allow-port-forwarding-in-nginx/22805067?noredirect=1#22805067), I managed to get a simple implementation working, so I thought I'd document it here.

Let's begin with Nginx. Here is the configuration file in it's simplest form. This is made available in `/etc/nginx/sites-enabled/elastic` (Debian/ Ubuntu):

    # Block default
    server {
            listen 80;
            return 301;
    }

    # Forward port <port_number> to elasticsearch
    server {

        listen *:<port_number>;

        location / {
            auth_basic "Restricted";
            auth_basic_user_file /var/data/nginx-elastic/.htpasswd;

            proxy_pass http://127.0.0.1:9200;
            proxy_read_timeout 90;
        }
    }

It's pretty obvious whats going on here. We 301 anything that hits the server on port 80. For requests made on `<port_number>`, we have enabled HTTP basic authentication which, upon success, will proxy to the Elasticsearch server. This can act as a good base for further configuration. We could, for example, require different levels of authentication for different Elasticsearch endpoints/ indices by providing additional location declartions. For example: `location ~ ^/.*/_search$`. Consult the [Nginx documentation](http://nginx.org/en/docs) for more information.

The `apache2-utils` Debian package was used to generate the HTTP credentials:

    $ sudo apt-get install apache2-utils
    $ sudo htpasswd -c /var/data/nginx-elastic/.htpasswd myusername

    # Enter password when prompted

    # Restart Nginx
    $ sudo service nginx restart

Check everything works so far:

    # Returns a 301
    $ curl <server_ip>

    # Returns 401 Authorization Required
    $ curl <server_ip>:<port_number>

    # Returns Elasticsearch output
    $ curl <username>:<password>@<server_ip>:<port_number>

The last thing to do is restrict Elasticsearch to only running on localhost; at the moment we could still access the server by hitting the port Elasticsearch runs on:

    $ curl <ip_address>:9200

To do this, edit `elasticsearch.yml` and change/ insert as necessary:

    network.host: 127.0.0.1
    http.host: 127.0.0.1

    # Restart Elasticsearch
    $ sudo service elasticsearch restart

This will force Elasticsearch to run on the local machine only; it will not be accessible from the wild.

Let's check that works:

    # Should fail/ connection refused
    $ curl <ip_address>:9200

That should be all you need for now. Of course you take this further by managing `iptables` to restrict ports even more if need be, but this is a good start!
