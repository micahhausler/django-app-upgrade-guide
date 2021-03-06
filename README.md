# Django 1.6 -> 1.7 Reusable App Migration Guide
A guide for upgrading 3rd party django apps from 1.6 to 1.7

## Table of Contents
- [Dependencies](#dependencies)
- [Settings](#settings)
- [App Config](#app-config)
- [Migrations](#migrations)
- [1.6 Support](#1.6-support)

## Dependencies
- Check `.travis.yml`/`circle.yml` and update django version to test against 1.7
- In `setup.py`:
  - Check that `install_requires=` includes `Django>=1.6.5`
  - Check that `tests_require=` includes `south>=1.0.2`
- in `setup.cfg`, add to the `[flake8]` directive's `exclude` option: `...,south_migrations`
- in `.coveragerc` add to the `[run]` directive's `omit` option: `<app-name>/south_migrations/` 

## Settings
In your `settings.py`, be sure to add 

```python
MIDDLEWARE_CLASSES=(),
```

## App Config
- Add an `app.py` in the app directory. See [django's docs](https://docs.djangoproject.com/en/1.7/ref/applications/#for-application-authors) for full information.

  ```python
  from django.apps import AppConfig


  class RockNRollConfig(AppConfig):
      name = 'rock_n_roll'
      verbose_name = 'Rock ’n’ roll'
  
  ```
 
- Update `__init__.py` in the app's base directory like so:

  ```python
  
  default_app_config = 'rock_n_roll.apps.RockNRollConfig'
  ```

## Migrations
If an app has a `migrations/` module, you'll need to follow these steps:
- Rename the `migrations/` module to `south_migrations/` (and remove any .pyc files)
- Install `django==1.7.7`
- Run `makemigrations` 

```bash
find . -name "*.pyc" | xargs rm
mv ./migrations/ ./south_migrations/
cd ..
python manage.py makemigrations <package_name>
```

## 1.6 Support
- If you plan on supporting 1.6 and 1.7, be sure to document the need for `south` in 1.6 versions.
- Dynamically add `south` to the `INSTALLED_APPS` setting like so:

  ```python
  import django
  if django.VERSION[1] < 7:
      installed_apps += 'south',
  # or
  ) + (('south',) if django.VERSION[1] <= 6 else ()),
  ```           
- Add to `run_tests.py`:

  ```python
  configure_settings()
  if django.VERSION[1] >= 7:
      django.setup()
  ```
