# Plan for Brook

## Hash Tables

Do as you would normally.
Explicitly mention these other bits:

* Relationship of Hash tables to Python dictionaries
* By default are of fixed size; dictionaries are flexible hash tables
* Benefit of lookup time for hash tables vs lists/trees
* Naming collisions and how to handle them: increase hash table size or have buckets to handle keys that map to the same hash

What are 5 tests that could be written for this type of data structure?

## If Code Review...

Review everything to date and have them explain every step of what they've done and what various things mean. 

## Django Exercise: To-Do List

Want to use Django to build a simple To Do list.

* Create a new virtual environment to isolate packages that will be downloaded from the rest of your system. Create a repository to put the project in version control. Get [the standard Python .gitignore](https://github.com/github/gitignore/blob/master/Python.gitignore) from GitHub 

```
$ python3 -m venv ENV
$ source ENV/bin/activate
(ENV) $ git init
(ENV) $ touch README.md
(ENV) $ touch .gitignore
```

* Commit the above to the repository.
* Build Django project scaffold to get started

```
(ENV) $ pip install django
(ENV) $ django-admin startproject django-tasklist
```

* Create a `requirements.txt` or `requirements.pip` file (they're the same ultimately) to keep an up-to-date record of the Python packages that the Django project needs in order to work.

```
(ENV) $ pip freeze > requirements.txt
```

* Commit `requirements.txt` to the repository
* Change directory into the `django-tasklist` directory. You'll know that you're in the project root when you see the `manage.py` file.
* Configure the settings to use a postgres database with configuration saved in environment variables
    - modify `settings.py` so that the database engine uses the postgres backend for Django instead of sqlite
    - create the postgres database to be used for this Django project
    - modify `settings.py` so that the database `NAME` receives the value of the `DATABASE_NAME` environment variable
    - add the `USER`, `PASSWORD`, `HOST`, and `PORT` key-value pairs, so that they each receive their values from environment variables but default to empty strings if the environment variables don't exist
    - add the environment variables that you just told Django it needed to the `ENV/bin/activate` file
    - re-activate the environment so that the environment variables become active in the current terminal session
    - add the `TEST` key to the `DATABASES` dictionary in `settings.py`, setting the name of the test database to be anything

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3' --> 'django.db.backends.postgresql',
        'NAME': os.path.join(BASE_DIR, 'db.sqlite3') --> os.environ.get('DATABASE_NAME', ''),
        # below here will be new
        'USER': os.environ.get('DATABASE_USER', ''),
        'PASSWORD': os.environ.get('DATABASE_PASSWORD', ''),
        'HOST': os.environ.get('DATABASE_HOST', ''),
        'PORT': os.environ.get('DATABASE_PORT', ''),
        'TEST': {
            'NAME': 'some_test_database'
        }
    }
}
```

* Apply the existing migrations from Django's built-in models to your database

```
(ENV) $ python manage.py migrate
```

* Commit changes up to this point to the repository
* Create a new Django app inside of the Django project called "tasks"

```
(ENV) $ python manage.py tasks
```

* Build model inside of `tasks/models.py`. Outline of a Task model:
    - title
    - category (choices of: work, school, personal); default to "personal"
    - complete (True, False); default to False
    - due date (optional)
    - string representation should be something like this:
        <Task [the text of the title] category: [category] complete: [True/False]>

```python
from django.db import models

CATEGORY_CHOICES = [
    ('W', 'work'), ('S', 'school'), ('P', 'personal')
]

class Task(models.Model):
    """There should be a doc string here."""

    title = models.CharField(max_length=255)
    category = models.CharField(choices=CATEGORY_CHOICES, max_length=2, default='p')
    complete = models.BooleanField(default=False)
    due_date = models.DateTimeField(null=True)

    def __repr__(self):
        output = "<Task {} category: {} complete: {}>"
        return output.format(self.title, self.category, self.complete)
```

* Add a model manager near the top of `tasks/models.py` to retrieve only incomplete tasks

```python
class TaskManager(models.Manager):
    """There should be a doc string here."""

    def get_queryset(self):
        return super(TaskManager, self).get_queryset().filter(complete=False)
```

* Add an instance of the new model manager to the `Task` model, as well as an instance of the default model manager so that you can access model instances for complete tasks and all tasks respectively.

```python
class Task(models.Model):
    # all of the other class attributes here
    incomplete_tasks = TaskManager()
    objects = models.Manager()

    # the __repr__ method goes below here
```

* Add the `tasks` app to the bottom of the `settings.py` file's list of `INSTALLED_APPS`
* Create migrations to prepare inserting this model's schema into the database

```
(ENV) $ python manage.py makemigrations
```

* Apply migrations to the database with `python manage.py migrate`
* Commit changes up to this point
* Write tests for the `Task` model in `tasks/tests.py`. Some tests to consider:
    - An instance of the `Task` model has the "title", "category", "complete", and "due_date" attributes
    - If a title is given for a new `Task` instance, the defaults are what they're supposed to be
    - The string representation appears with the exact format outlined in the model definition
    - You can use the `incomplete_tasks` model manager on the `Task` class object to retrieve *only* incomplete tasks, instead of what `objects` does which is return all tasks
* Run the above tests with `python manage.py test`
* Commit the tests written
* Add a `urlpattern` to `django-tasklist/urls.py` for a route named `"home"`. The pattern itself should simply resolve to `"/"`, and should use a view called `home_view` imported from `django_tasklist.views`

```python
# other stuff above this
from django_tasklist.views import home_view

urlpatterns = [
    url(r'^$', home_view, name='home'),
    url(r'^admin/', admin.site.urls)
]
```

* Create a `views.py` file in `django-tasklist` and add a `home_view` to that file that takes in a `request` object and renders to a template called `django-tasklist/home.html`.

```python
from django.shortcuts import render

def home_view(request):
    return render(request, "django-tasklist/home.html")
```

* Create the directory for the templates as `django-tasklist/templates/django-tasklist`
* In `django-tasklist/templates/django-tasklist` create `layout.html` and `home.html`
    - `home.html` should inherit from `base.html` and only fill in a content block
    - `base.html` should just be a basic HTML layout with a content block to fill with data from any other template
* Commit the changes made thus far.
* The `home_view` should provide to the template two key-value pairs:
    - a list of incomplete tasks
    - a list of complete tasks

```python
from django.shortcuts import render
from tasks.models import Task

def home_view(request):
    incomplete = Task.completed_tasks.all()
    complete = Task.objects.filter(complete=True).all()
    context = {'incomplete': incomplete, 'complete': complete}
    return render(request, "django-tasklist/home.html", context=context)
```

* Refactor `home.html` to have the list of incomplete tasks in a div to the left, and the complete tasks in a div to the right with appropriate headings
* Commit the changes
* Write tests for the view and home route.
* Commit the tests

If there's time, have the class add the following:

- A view and template in the `tasks` app for adding new tasks, which serves on the route `/tasks/add`
- A view and template in the `tasks` app for listing tasks in a certain category, which serves on the route `/tasks/category/(?P<cat>\w+)`
- A view and template in the `tasks` app for listing only tasks with a due date on the route `/tasks/due_dates`
