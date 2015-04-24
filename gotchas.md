# Django 1.7 Gotchas

### Contents
- [Management Commands](#management_commands)
- [Custom User Model](#custom_user_model)
- [Imports](#imports)
- [Models](#models)
- [New Migrations](#new_migrations)
	- [Schema Migrations](#schema_migrations)
	- [Data Migrations](#data_migrations)


## Management Commands
- `manage.py migrate` no longer takes the `--all` option. It is default
- The `--noinput` option has replaced `--interactive=False` on management commands
  - When using `call_command()`, still use the `interactive=False` kwarg, as `interactive` is the 'destination' for `noinput`


## Custom User Model
In Django 1.7, we've upgraded to a custom user model, and you can see this in Ambition at `ambition/core/user/models.py`. Initially it is the exact same as the default user, but with a username max_length of 255. This is because we had used the `longerusername` package to alter our database schema, but that will no longer work with Django 1.7.

Almost everywhere[\[1\]][1] you would have used `from django.contrib.auth.models import User`, substitue `from django.contrib.auth import get_user_model`. See the [documentation][1] for usage with signals that rely on user actions. For ForeignKeys to a user, use `settings.AUTH_USER_MODEL`. 

[1]: https://docs.djangoproject.com/en/1.8/topics/auth/customizing/#referencing-the-user-model  "Referencing the User model"

Example:

```python
from django.conf import settings
#from django.contrib.auth.models import User
from django.contrib.auth import get_user_model
from django.db import models

class SomeModel(models.Model):

	user = models.ForeignKey(settings.AUTH_USER_MODEL)

	def find_user(self, username):
		return get_user_model().objects.filter(username=username).first()
	
	def create_admin(self, username, password):
		return get_user_model().objects.create_superuser(username=username, password=password)
```

Our custom user model inherits or implements everything the built-in Django user model does, so you can do user operations like normal.

```python
from django.contrib.auth import get_user_model

get_user_model().objects.create_superuser(...)
get_user_model().objects.filter(is_staff=True).last()
```

## Imports
Imports for Django 1.7 MUST be either absolute path or relative path

```python
# NOT THIS - this will error out
from myapp.models import MyModel

# Must be absolute path or relative
from myproject.apps.myapp.models import MyModel
from .models import MyModel
```

## Models

### Creation time on model field
If you set a field default to `datetime.utcnow()` the migration detector will detect a change between your initial migration and the evaluation of `datetime.utcnow()` and make a new migration. THIS IS BAD.

```python
# Migration 0001_initial
('last_partial_computation_time', models.DateTimeField(default=datetime.datetime(2015, 3, 30, 21, 19, 39, 940338)))

# Migration 0002_auto_yyyymmdd_hhmm.py
migrations.AlterField(
    model_name='ambitionscorecomputationconfig',
    name='last_partial_computation_time',
    field=models.DateTimeField(default=datetime.datetime(2015, 4, 18, 15, 12, 48, 695409)),
    preserve_default=True,
)
# BAD NEWS BEARS
```
The solution is to set the field to `default=datetime.utcnow` or `auto_now=True`

```python
# NOT THIS
models.DateTimeField(default=datetime.utcnow())

# Set to `datetime.utcnow` or `auto_now=True`
models.DateTimeField(default=datetime.utcnow)
```


### Choices on CharField must be in a consistent order
If you do not have a consistent order for your `choices` kwarg on a CharField, the migration detector may make a new automatic migration with a different order.

```python
# NOT THIS
models.CharField(choices=list(some_dict.items()), max_length=255)

# Must be in consistent order
def get_choices():
	some_chioces = tuple(some_dict.items())
	some_choices.sort()

models.CharField(choices=get_choices(), max_length=255)
```

## New Migrations
I highly reccomend reading [Django's docs](https://docs.djangoproject.com/en/1.7/topics/migrations/) on migrations, but here is the quick and dirty summary.

### Schema migrations
If you have to perform a schema change, 1.7 migrations are really simple.

1. Create/alter your model.
1. Run `python manage.py makemigrations <app_name>` 
    
    **Note:** you only need to specify the app_name, not the full dotted path.
1. Run `python manage.py migrate`
1. Boom. You're done

### Data migrations
Data migrations are not more difficult than they were with South. Read the [Django docs](https://docs.djangoproject.com/en/1.7/topics/migrations/#data-migrations) to get a full picture of what your options are. You can see an example of a custom SQL migration in ambition at `ambition/core/collection/migrations/0002_auto_20150413_1903.py`

