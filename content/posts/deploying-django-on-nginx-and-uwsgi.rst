Deploying Django on Nginx and uWSGI
###################################

:date: 2013-04-07
:author: Jin Zhang

Nginx and uWSGI has been growing in popularity in the Django community due to `benchmarks <http://nichol.as/benchmark-of-python-web-servers>`_ showing them to be faster then the standard Apache or Gunicorn setup. I recently went through the process of setting up a fresh VPS with this setup and decided to document my process here.

I will be using Ubuntu 12.04 x64 Server edition. First let's install the packages we will need for this project.

.. code-block:: python

    $ sudo apt-get update
    $ sudo apt-get upgrade
    $ sudo apt-get install build-essential python-dev
    $ sudo apt-get install python-pip git
    # Required for uWSGI
    $ sudo apt-get install libxml2-dev
    # Install nginx and uWSGI
    $ sudo apt-get install nginx
    $ sudo pip install virtualenv
    $ sudo pip install uwsgi

We will use Upstart to automatically start up uWSGI on restarts. Create the file :code:`/etc/init/uwsgi.conf` and insert:

.. code-block:: python

    description     "uWSGI Emperor"

    start on runlevel [2345]
    stop on runlevel [06]

    respawn

    env LOGTO=/home/web/logs/uwsgi/emperor.log
    env BINPATH=/usr/local/bin/uwsgi

    exec $BINPATH --master --emperor /etc/uwsgi/apps-enabled --die-on-term --uid www-data --gid www-data --logto $LOGTO

Nginx and uwsgi will be running as user www-data and group www-data so we need to give them permissions to our web and logs directory. I will place these at :code:`/home/web/www` and :code:`/home/web/logs`. First we will add our user :code:`web` to the www-group.

.. code-block:: python

    $ sudo adduser web www-data
    $ mkdir www
    $ sudo chgrp www-data /home/web/www
    # Ensure future file additions inherit the same permissions
    $ sudo chmod g+rwxs /home/web/www
    $ mkdir logs
    $ sudo chgrp www-data /home/web/logs
    $ sudo chmod g+rwxs /home/web/logs
    $ mkdir /home/web/logs/uwsgi

Create a directory structure similar to the one below to hold our project. The config directory will hold our nginx and uwsgi configs for this specific website. The django directory holds our django project files. The logs directory contains the logs specific for the website.com site and the venv contains our virtual environment files.

.. code-block:: python

    /home/web/www/website.com/
    --------------------- config/
    --------------------- django/
    --------------------- logs/
    --------------------- venv/

Now create our uWSGI config file in :code:`/website.com/config/website.ini`

.. code-block:: python

    [uwsgi]
    ; define variables to use in this script
    project = django
    base_dir = /home/web/www/website.com
    uid = www-data
    gid = www-data
    ; process name for easy identification in top
    procname = %(project)
    ; number of worker processes
    processes = 2
    ; project-level logging to the logs/ folder
    logto = %(base_dir)/logs/uwsgi.log
    ; django >= 1.4 project
    chdir = %(base_dir)/%(project)
    module = %(project).wsgi
    ; add virtual environment to path
    virtualenv = %(base_dir)/venv
    ; unix socket (referenced in nginx configuration)
    socket = /home/web/www/website.com/django.sock
    chmod-socket = 666
    ; run master process as root
    master = true
    master-as-root = true

Also create our nginx config file in :code:`/website.com/config/website.conf`

.. code-block:: python

    server {
        listen       80;
        server_name  website.com www.website.com;

        root        /home/web/www/website.com/;
        access_log  /home/web/www/website.com/logs/nginx_access.log;
        error_log   /home/web/www/website.com/logs/nginx_error.log;

        location /static/ {
        alias /home/web/www/website.com/django/django/static/;
        expires 30d;
        access_log off;
        }

        location / {
            include uwsgi_params;
            uwsgi_pass unix:/home/web/www/website.com/django.sock;
        }
    }

Now we need to enable our site and to do that we simply link it to the directory nginx and uWSGI are searching for our sites.

.. code-block:: python

    # Create the directory to hold our uwsgi active apps
    $ sudo mkdir -p /etc/uwsgi/apps-enabled
    # uWSGI linking
    $ sudo ln -s /home/web/www/website.com/config/website.ini /etc/uwsgi/apps-enabled/website.ini
    # nginx linking
    $ sudo ln -s /home/web/www/website.com/config/website.conf /etc/nginx/sites-enabled/website.conf

Restart nginx and uwsgi.

.. code-block:: python

    $ sudo service nginx restart
    $ sudo service uwsgi restart

If you get an error about server_names_hash_bucket_size, this means our server name structure is a bit long for nginx. To fix this, simply uncomment the line below in :code:`/etc/nginx/nginx.conf`

.. code-block:: python

    Restarting nginx: nginx: [emerg] could not build the server_names_hash, you should increase server_names_hash_bucket_size: 32
    #
    # At /etc/nginx/nginx.conf, uncomment the line below then retry the nginx restart:
    server_names_hash_bucket_size 64;

Next we will create a empty Django project to test our server.

.. code-block:: python

    $ cd /home/web/www/website.com
    # Create the virtualenv
    $ virtualenv venv --distribute
    $ source venv/bin/activate
    # Now lets create our Django project
    $ pip install django
    $ venv/bin/django-admin.py startproject django

Now browse to your server address and you should see the Django welcome page. Congratulations, you have just deployed a production ready Django project on nginx and uWSGI!