# Deploying a Django application

In this tutorial we're going to show you how to deploy a Django application on [cloudControl]. You can find the complete [source code][example-app] of the application on Github, which is the official [Django tutorial]. The application allows to create, use and manage simple polls.

## The Django application explained

### Get the App

First, clone the Django application from our repository:

~~~bash
$ git clone git://github.com/cloudControl/python-django-example-app.git
$ cd python-django-example-app
~~~

### Optional: Start the App Locally Using Built-In Server

Install Django:

~~~bash
$ sudo pip install Django==1.4.3
~~~

Prepare the database by running:

~~~bash
$ python manage.py syncdb
~~~

When asked, create an admin user.

Finally, run the server locally to make sure that the app is working:

~~~bash
$ python manage.py runserver
~~~

Now you can access [/polls](http://localhost:8000/polls/) or [/admin](http://localhost:8000/admin/) to test the app locally. It's time to prepare it for deployment on our platform.

### Managing dependencies

The [python buildpack] uses [pip] to manage dependencies. Create a `requirements.txt` file with the following content:

~~~
Django==1.4.3
~~~

### Production server

In a production environment you normally don't want to use the development server. In this tutorial you are going to use [gunicorn] as the production server. To do so, add the following line to the `requirements.txt` file:

~~~
gunicorn==0.17.2
~~~

And finally add `gunicorn` to the list of installed applications (`INSTALLED_APPS` in `mysite/settings.py`):

~~~python
INSTALLED_APPS = (
    ...
    'gunicorn'
    ...
)
~~~

### Defining the process type

cloudControl uses a [Procfile] to know how to start your processes. Create a file called `Procfile` with the following content:

~~~
web: python manage.py run_gunicorn -b 0.0.0.0:$PORT
~~~

The `web` process type is required and specifies the command that will be executed when the app is deployed.

### Production database

Now it's time to configure the production database. The application currently uses SQLite as the database in all environments, even the production one. It is not possible to use a SQLite database on cloudControl because the filesystem is [not persistent][filesystem].

To use a database, you should choose an Add-on from [the Data Storage category][data-storage-addons]. Let's use the [Shared MySQL Add-on][mysqls].

To do so, update `requirements.txt` to install MySQL driver:

~~~
Django==1.4.3
gunicorn==0.17.2
MySQL-python==1.2.4
~~~

Next, modify the `mysite/settings.py` file to [get the MySQL credentials][get-conf] when running
on the platform:

~~~python
# Django settings for mysite project.

import os
import json

PROJECT_ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))

try:
    # production settings
    f = os.environ['CRED_FILE']
    db_data = json.load(open(f))['MYSQLS']

    db_config = {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': db_data['MYSQLS_DATABASE'],
        'USER': db_data['MYSQLS_USERNAME'],
        'PASSWORD': db_data['MYSQLS_PASSWORD'],
        'HOST': db_data['MYSQLS_HOSTNAME'],
        'PORT': db_data['MYSQLS_PORT'],
    }
except KeyError, IOError:
    # development/test settings:
    db_config = {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': '{0}/mysite.sqlite3'.format(PROJECT_ROOT),
    }
...
~~~

Update database configuration:
~~~python
DATABASES = {
    'default': db_config,
}
~~~

Commit all changes:

~~~bash
$ git add .
$ git commit -am "Migrate to cloudControl"
~~~

## Pushing and Deploying your App

Choose a unique name to replace the `APP_NAME` placeholder for your application and create it on the cloudControl platform: 

~~~bash
$ cctrlapp APP_NAME create python
~~~

Push your code to the application's repository, which triggers the deployment image build process:

~~~bash
$ cctrlapp APP_NAME/default push
Counting objects: 31, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (25/25), done.
Writing objects: 100% (31/31), 7.11 KiB, done.
Total 31 (delta 3), reused 24 (delta 0)

-----> Receiving push
-----> Preparing Python interpreter (2.7.2)
-----> Creating Virtualenv version 1.7.2
       New python executable in .heroku/venv/bin/python2.7
       Also creating executable in .heroku/venv/bin/python
       Installing distribute......done.
       Installing pip................done.
       Running virtualenv with interpreter /usr/bin/python2.7
-----> Activating virtualenv
-----> Installing dependencies using pip version 1.2.1
       Downloading/unpacking Django==1.4.3 (from -r requirements.txt (line 1))
       ...
-----> Building image
-----> Uploading image (8.6M)

To ssh://APP_NAME@cloudcontrolled.com/repository.git
 * [new branch]      master -> master
~~~

Add MySQLs Add-on with `free` plan to your deployment and deploy it:
~~~bash
$ cctrlapp APP_NAME/default addon.add mysqls.free
$ cctrlapp APP_NAME/default deploy
~~~

Finally, prepare the database using the [Run command][ssh-session] (when prompted create admin user):

~~~bash
$ cctrlapp APP_NAME/default run "python manage.py syncdb"
~~~

You can login to the admin console at `APP_NAME.cloudcontrolled.com/admin`, create some polls and see them at `APP_NAME.cloudcontrolled.com/polls`.

For additional information take a look at [Django Notes][django-notes] and other [python-specific documents][python-guides].

[django]: https://www.djangoproject.com/
[cloudControl]: http://www.cloudcontrol.com
[cloudControl-doc-user]: https://www.cloudcontrol.com/dev-center/Platform%20Documentation#user-accounts
[cloudControl-doc-cmdline]: https://www.cloudcontrol.com/dev-center/Platform%20Documentation#command-line-client-web-console-and-api
[Procfile]: https://www.cloudcontrol.com/dev-center/Platform%20Documentation#buildpacks-and-the-procfile
[git]: https://help.github.com/articles/set-up-git
[filesystem]: https://www.cloudcontrol.com/dev-center/Platform%20Documentation#non-persistent-filesystem
[data-storage-addons]: https://www.cloudcontrol.com/dev-center/Add-on%20Documentation/Data%20Storage/
[mysqls]: https://www.cloudcontrol.com/dev-center/Add-on%20Documentation/Data%20Storage/MySQLs
[example-app]: https://github.com/cloudControl/python-django-example-app
[django-notes]: https://www.cloudcontrol.com/dev-center/Guides/Python/Django%20notes
[get-conf]: https://www.cloudcontrol.com/dev-center/Guides/Python/Add-on%20credentials
[Django tutorial]: https://docs.djangoproject.com/en/1.4/intro/tutorial01/
[python-guides]: https://www.cloudcontrol.com/dev-center/Guides/Python
[python buildpack]: https://github.com/cloudControl/buildpack-python
[pip]: http://www.pip-installer.org/
[gunicorn]: http://gunicorn.org/
[worker]: https://www.cloudcontrol.com/dev-center/Platform%20Documentation#scheduled-jobs-and-background-workers
[db-commit]: https://github.com/cloudControl/python-django-example-app/commit/983f45e46ce0707476cec167ea062e19adcb53c9
[ssh-session]: https://www.cloudcontrol.com/dev-center/Platform%20Documentation#secure-shell-ssh