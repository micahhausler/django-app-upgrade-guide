# Dropping Django 1.6, add Django 1.8 support 
Since dropping 

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
  - find . -name 'south_migrations' -type f
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

- in `settings.py` in the INSTALLED_APPS setting:

```diff
@@ -1,6 +1,5
  import os
- import django
  from django.conf import settings

@@ later def configure_settings():

-            ) + (('south',) if django.VERSION[1] <= 6 else ()),
+            ),
             ROOT_URLCONF='data_schema.urls',
             DEBUG=False,
```

- In `setup.py`
  - Remove classifiers for `Framework :: Django :: 1.6` and add 1.8
  - In `install_requires=[]` increase Django to `Django>=1.7`
  - In `tests_require=[]` increase django-nose to `django-nose>=1.4`
  - **Remove `south` from `install_requires` or `tests_require`**

## Tests

Ok this is annoying. In 1.8, it is not possible to assign a ForeignKey relation in memory without the ForeignKey model being saved. So this test syntax is no longer viable:

```python
from django_dynamic_fixture import N
from django.contrib.auth.models import Group
from django.contrib.auth import get_user_model
from django.test import TestCase

class MyTestClass(TestCase):
	def setUp(self):
	    super(MyTestClass, self).setUp()

		self.group = N(name='jedi')
		self.user = N(get_user_model(), username='lukeskywalker', groups=[self.group])


$ python manage.py test
ValueError: Cannot assign "<Group: jedi>": "Group" instance isn't saved in the database
```
We _can_ subclass ForeignKey, set [`allow_unsaved_instance_assignment=True`](https://docs.djangoproject.com/en/1.8/ref/models/fields/#django.db.models.ForeignKey.allow_unsaved_instance_assignment), and replace it everywhere we use a ForeignKey, but that is pretty obnoxious. The solution is to replace
django_dynamic_fixture's `N` function with `G`.

