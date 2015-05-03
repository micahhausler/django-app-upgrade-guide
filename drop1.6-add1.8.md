# Dropping Django 1.6, add Django 1.8 support

First off, take a minute to read the [Django 1.8 Release Notes](https://docs.djangoproject.com/en/1.8/releases/1.8/)

Second, read the new [Postgres contrib package](https://docs.djangoproject.com/en/1.8/ref/contrib/postgres/) documentation
 
## Outline
- [Fix typo](#fix-typo)
- [Checklist](#checklist)
- [Importing modules in `__init__.py`](#importing-models-in-__init__py)
- [Argparse](#optparse-argparse)
- [Tests](#tests)

## Fix Typo
I accidentally edited the `__init__.py` of all of our django projects to have
`django_app_config =` where it should have `default_app_config =`. Could you replace that where you find it?
 

##Checklist:

- In `.coveragerc` remove south_migrations from `omit` directive 

```diff
 omit =
     */migrations/*
-    */south_migrations/*
     */version.py
```

- Change versions in `.travis.yml`
  - Use `>=1.8,<1.9` so that `1.8` is included
  - Be sure django-nose is `>=1.4`

```diff
   matrix:
-    - DJANGO=1.6.11
-    - DJANGO=1.7.7
+    - DJANGO=">=1.7,<1.8"
+    - DJANGO=">=1.8,<1.9"
 install:
-  - pip install -q coverage flake8 Django==$DJANGO django-nose
+  - pip install -q coverage flake8 Django$DJANGO django-nose>=1.4
```

- Remove any `south_migrations` directories
  - `find . -name 'south_migrations' -type f | xargs rm -rf`
- Check that `requirements/docs.txt` has `Django>=1.7`
- In `run_tests.py`:

```diff
 import django
-from django.conf import settings
 
 from settings import configure_settings
 
 @@ -20,10 +19,6 @@
 
 
 def run_tests(*test_args, **kwargs):
-    if 'south' in settings.INSTALLED_APPS:
-        from south.management.commands import patch_for_test_db_setup
-        patch_for_test_db_setup()
```

And later remove:

```diff
-    parser.add_options(NoseTestSuiteRunner.options)
```


- in `settings.py` in the INSTALLED_APPS setting:

```diff
@@ -1,6 +1,5
  import os
- import django
  from django.conf import settings

@@ later def configure_settings():

-            ) + (('south',) if django.VERSION[1] <= 6 else ()),
+            ),
+            TEST_RUNNER='django_nose.NoseTestSuiteRunner',
+            NOSE_ARGS=['--nocapture', '--nologcapture', '--verbosity=1'],
             ROOT_URLCONF='data_schema.urls',
             DEBUG=False,
```

- In `setup.py`
  - Remove classifiers for `Framework :: Django :: 1.6` and add 1.8
  - In `install_requires=[]` increase Django to `Django>=1.7`
  - In `tests_require=[]` increase django-nose to `django-nose>=1.4`
  - **Remove `south` from `install_requires` or `tests_require`**

## Importing models in `__init__.py`
If you can avoid importing anything into `django-someapp/__init__.py`, please do. 
If you import your models into here or import another file that imports your models, you'll need to sepcify an `app_lable = 'app_name'` on each of your models. This is a Django 1.9 change but 1.9 is coming out Fall 2015, and 1.8 will print a warning if you don't do this.

Also, try running your tests like this to find any 1.9 errors:
```
python -Wall manage.py test
```

Here is what you need to add if you import your models in your `__init__.py`

```diff
@@ -11,5 +11,9 @@ class DBMutex(models.Model):
     :type creation_time: datetime
     :param creation_time: The creation time of the mutex lock
     """
+
+    class Meta:
+        app_label = 'db_mutex'
+
```

## ~~optparse~~ argparse
[Django switched its argument parser](https://docs.djangoproject.com/en/1.8/releases/1.8/#management-commands-that-only-accept-positional-arguments) from the depricated [optparse](https://docs.python.org/2/library/optparse.html) to [argparse](https://docs.python.org/3/library/argparse.html). 
If you've added custom arguments to a management command, read through the [updated 1.8 docs](https://docs.djangoproject.com/en/1.8/howto/custom-management-commands/) on adding arguments.


## Tests

Ok this is annoying. In 1.8, it is not possible to assign a ForeignKey relation in memory without the ForeignKey model being saved. So this test syntax is no longer viable:

```python
from django_dynamic_fixture import N
from django.contrib.auth.models import Group
from django.contrib.auth import get_user_model
from django.test import TestCase

class StarWarsTestClass(TestCase):
	def setUp(self):
	    super(StarWarsTestClass, self).setUp()

		self.group = N(name='jedi')
		self.user = N(get_user_model(), username='lukeskywalker', groups=[self.group])


$ python manage.py test
ValueError: Cannot assign "<Group: jedi>": "Group" instance isn't saved in the database
```
We _can_ subclass ForeignKey, set [`allow_unsaved_instance_assignment=True`](https://docs.djangoproject.com/en/1.8/ref/models/fields/#django.db.models.ForeignKey.allow_unsaved_instance_assignment), and replace it everywhere we use a ForeignKey, but that is pretty obnoxious. The solution is to replace
django_dynamic_fixture's `N` function with `G`.

