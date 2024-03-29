# CLE.py Tutorial: Django + Postgres + Celery + Redis

## Prerequisites
Install [Docker](https://docs.docker.com/install/)

Note: Windows 10 version requires Pro, Enterprise or Education editions. For Home edition use [Docker Toolbox](https://docs.docker.com/toolbox/overview/)


## Getting to it
Fork or clone the tutorial repo:

https://github.com/rsolano/clepy-docker-compose

Clone command:

`git clone https://github.com/rsolano/clepy-docker-compose`

## 1. Define an image to use as our Python runtime container.

Edit the `Dockerfile` so that it pulls from a Python image. You can use `python:3.7` here.

## 2. Define app services

Edit the `docker-compose.yml` in order to define our app services:


* `postgres` The database. Use the `postgres:alpine` image.


* `redis` We'll use this as a message broker to communicate with Celery. Use the `redis:alpine` image.


* `web` Our Django image defined in Dockerfile. Use `python manage.py runserver 0.0.0.0:8000` as the `command` for the container to run.


* `worker` Celery, our task worker. Use `celery -A tutorial worker -l info` as the `command` for the container to run.


## 3. Create Django project inside the container

Run:

`docker-compose run web django-admin startproject tutorial .`

## 4. Start the application
Run:

`docker-compose up -d`

The `-d` option runs the services in the background.

### NOTE: Docker Toolbox only
If you are using Docker Toolbox, you need to start the VM, note its IP address and add it to the `ALLOWED_HOSTS` property in `settings.py`. Use this IP instead of `localhost` to access the application.

```
docker-machine start
docker-machine ip
```

To view the application logs:

`docker-compose logs -f`

The `-f` option follows the log output. 

You can also specify which service(s) to display logs for:

`docker-compose logs -f web worker`


## 5. Set up the database in Django
Edit the `tutorial/settings.py` file and modify the `DATABASES` section to point to our PostgreSQL instance.

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get('DB_NAME'),
        'USER': os.environ.get('DB_USER'),
        'PASSWORD': os.environ.get('DB_PASSWORD'),
        'HOST': os.environ.get('DB_HOST'),
        'PORT': '5432',
    }
}
```

### Initial migration
This will create the initial database schema in our PostgreSQL database:

`docker-compose exec web python manage.py migrate`

### Verifying database setup
Log into the `db` container:

`docker-compose exec db bash`

Switch to user `postgres`

`su postgres`

Use the `psql` utility's `\dt` command to view the tables created by Django:
```
psql

\dt
```

To exit the utility enter `\q`.

To exit the container enter `exit` twice.

You can now delete the default `db.sqlite3` database file which comes with Django.

The Django app should now be available at http://localhost:8000 at this point.

## 6. Configure Celery and create/run a task

First, move the `celery.py` file provided into the `tutorial/` directory in order to set up Django to use Celery.

In `tutorial/__init__.py`, add the following lines to ensure Django starts Celery:

```
from .celery import app as celery_app

__all__ = ('celery_app',)
```

Now create a Django app to hold our task logic:

`docker-compose exec web python manage.py startapp myapp`

Add the newly created app to the `INSTALLED_APPS` property in `settings.py`:

```python
INSTALLED_APPS = [
    'myapp.apps.MyappConfig',
    ...
]
```


Create a `myapp/tasks.py` file with the following code:

```python
import time
from random import random

from celery import task


@task
def perform_lengthy_task():
    print("Beginning time-consuming task...")
    time.sleep(random() * 10)
    print("Done!")
    return True
```

Now we need to restart the `worker` since our changes there won't auto-reload like Django does.

`docker-compose restart worker`

To run the task, open a Django shell:

`docker-compose exec web python manage.py shell`

Import the task and invoke using `delay()` in order for Celery to run it:

```
from myapp.tasks import *
perform_lengthy_task.delay()
```

The output should be an `AsyncResult` object which indicates the task was sent to Celery successfully.

```bash
<AsyncResult: 2dfd7bc5-c46c-4d21-ae23-46eebc933186>
```

