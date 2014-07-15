Installing Django on Heroku with Postgresql
###########################################

:date: 2013-04-06
:author: Jin Zhang

We will set up a basic Django site on the popular hosting platform `Heroku`_. The main benefit of Heroku is allowing developers to focus on writing code that differeniates their product and leaving the system administrator tasks to Heroku. They offer a free plan which is sufficient for small projects and personal blogs. The downside is that they do not allow custom domains under the free plan, instead you will be given a subdomain under heroku.com.


The following tutorial assumes you are running Linux Ubuntu or derivatives of it. First sign up for a free `Heroku`_ account and install the Heroku Toolbelt. The toolbelt allows us to run heroku's terminal commands and also installs the git revision control program.

.. code-block:: python

    $ wget -qO- https://toolbelt.heroku.com/install-ubuntu.sh | sh

Next let's install python pip and virtualenv. A virtualenv allows us to keep project dependencies separate from other projects.

.. code-block:: python

    $ sudo apt-get install python-pip
    $ pip install virtualenv

Create a directory called django_heroku to hold our django project and inside create a virtualenv called venv.

.. code-block:: python

    $ mkdir django_heroku
    $ cd django_heroku
    $ virtualenv venv --distribute

Now we need to activate the virtualenv.

.. code-block:: python

    $ source venv/bin/activate

We can tell that our virtualenv is active by the prefix applied to our shell :code:`(venv)`. Now let's install the dependencies we will need for the project. Psycopg2 is the python postgres database driver and dj-database-url allows us to use database URLs to configure our Django project which we will need for Heroku.

.. code-block:: python

    $ pip install django psycopg2 dj-database-url

Now create a new Django project called django_heroku. The period . at the end tells Django to start our new project in the current directory.

.. code-block:: python

    $ venv/bin/django-admin.py startproject django_heroku .

Test that our django project works with the built-in development server.

.. code-block:: python

    $ python manage.py runserver

    Validating models...

    0 errors found
    March 29, 2013 - 13:45:04
    Django version 1.5.1, using settings 'django_heroku.settings'
    Development server is running at http://127.0.0.1:8000/
    Quit the server with CONTROL-C.

Point your browser to http://127.0.0.1:8000 to see the Django welcome page. Now that our website is working, let's deploy it to heroku. Create a requirements.txt file to tell Heroku what dependencies we are using.

.. code-block:: python

    $ pip freeze > requirements.txt
    $ cat requirements.txt

    Django==1.5.1
    argparse==1.2.1
    distribute==0.6.24
    dj-database-url==0.2.1
    psycopg2==2.4.6
    wsgiref==0.1.2

Next we will need to configure settings.py to work with Heroku's database setting. Open up the settings.py file and add the following code to the bottom of the file.

.. code-block:: python

    # Set Django's database settings to Heroku's environment variable DATABASE_URL or
    # default to Sqlite if unable to find
    import os
    import dj_database_url
    basedir = os.path.abspath(os.path.dirname(__file__))
    DATABASES['default'] = dj_database_url.config(default=os.environ.get(
        "DATABASE_URL", "sqlite:///" + os.path.join(basedir, "database.db")))

We need to create two additional files, Procfile and .gitignore, for our project. The file directory should look similar to below:

.. code-block:: python

    /django_heroku/
    -- django_heroku/
    ----- __init__.py
    ----- settings.py
    ----- urls.py
    ----- wsgi.py
    -- venv/
    -- manage.py
    -- requirements.txt
    -- .gitignore
    -- Procfile

Create a file called Procfile to declare how Heroku should launch our project.

.. code-block:: python

    web: python manage.py runserver 0.0.0.0:$PORT --noreload

Create a file called .gitignore to tell git which files to ignore.

.. code-block:: python

    *.py[cod]
    # PyCharm
    .idea
    # Database
    *.db
    # Virtualenv
    venv

Commit our files to git. If this is your first time running git, you may need to set some global variables first.

.. code-block:: python

    $ git config --global user.name "John Doe"
    $ git config --global user.email johndoe@example.com


.. code-block:: python

    $ git init
    $ git add .
    $ git commit -m 'first commit'

Finally we are ready to deploy to Heroku.

.. code-block:: python

    $ heroku login
    $ heroku create
    $ git push heroku master

If Heroku tells you to make a ssh key, create one and save it to the default location which should be similar to :code:`/home/YOUR_USERNAME/.ssh/id_rsa`

Visit the URL that Heroku gave you and you should see the Django welcome page! Congratulations, you have just launched a working Django site to Heroku. Now grab a coffee and take a well deserved break.


.. _Heroku: http://heroku.com
