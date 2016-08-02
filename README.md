# GeoDjango Tutorial (using OSX, PostgreSQL.app bundled with PostGIS, and bootstrapped for development on github)

This tutorial will go over how to setup Macintosh OSX (10.11.16) for developing a  [GeoDjango](https://docs.djangoproject.com/en/dev/ref/contrib/gis/tutorial/) web application.

GeoDjango requires a geo-spatial SQL database. There are several excellent options to choose from, such as: MySQL, SQLite, Oracle, and PostgreSQL.

PostgreSQL server with PostGIS was chosen `because it is the most mature and feature-rich open source spatial database.` [1](https://docs.djangoproject.com/en/dev/ref/contrib/gis/install/#spatial-database) The SpatialLite library for SQLite did seem intriguing as a light-weight, embedded server. There are inconsistencies between the way SQL is structured between PostGIS and SpatialLite, so it does bring up the question on how well the Django ORM abstracts these inconsistencies from the developer... Regardless, PostgreSQL is what will be used in the majority of production environments.

## Postgres Installation
The recommended way to install [PostgreSQL](https://www.postgresql.org) and [PostGIS](http://postgis.net) on OSX is to use the bundled [Postgres.app](http://postgresapp.com).

1. Download and install the [Postgres.app](http://postgresapp.com)

2. Setup your PATH

  ```
  vi ~/.bash_profile
  export PATH=$PATH:/Applications/Postgres.app/Contents/Versions/latest/bin
  ```

3. Create the PostgreSQL user and Django application database

  ```
  CREATE DATABASE geodjango;
  CREATE USER geo WITH PASSWORD 'geo';
  GRANT ALL PRIVILEDGES ON DATABASE geodjango to geo;
  ALTER ROLE geo SUPERUSER;
  ```

  Note: the role `SUPERUSER` is required in order to run `CREATE EXTENSION postgis;`. This command is run during the migrate process and replaces the need for using `template_postgis` seen in older tutorials.

## Python Virtualenv

Its useful to have each python environment separated for development, and to keep the OSX sytsem `python2.7/site-packages` clean. One widely used method is to install [homebrew](http://brew.sh) and [virtualenv](https://virtualenv.pypa.io/en/stable/).

1. Install homebrew (and Python 2.7)

  ```
  /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
  brew install python
  ```

2. Install virtualenv

  Note: `which python` should report back with `/usr/local/bin/python`
  
  Install virtualenv using `pip` to the homebrew Python interpreter site-packages so it will be available 'globally'.
  
  ```
  pip install virtualenv
  ```

3. Bootstrap Project

  Since we want to keep Python site-packages clean, lets bootstrap Django by creating the project directory `geodjango` and the virtual environment.
  
  ```
  mkdir geodjango && cd geodjango/
  virtualenv .vpython
  source .vpython/bin/activate
  ```

4. Install dependencies

  Make a text file called `requirements.txt` with the following contents:
  
  ```
  django==1.9.8
  psycopg2==2.6.2
  configparser==3.5.0
  filelock==2.0.6
  ```

  Install the dependencies using pip.
  ```
  pip install -r requirements.txt
  ```

5. Initialize git

  Create a text file called '.gitignore' with the following contents:
  
  ```
  .DS_Store
  *.pyc
  *.log
  *.pot
  .vpython/
  environment.ini
  environment.ini.lock
  ```

  Initialize the git repository:
  
  ```
  git init
  ```

## Django project

1. Start the Django project

  ```
  python .vpython/bin/django-admin startproject geodjango .
  ```
  
  The file structure should be as follows:
  
  ```
  geodjango/
    .vpython/
    geodjango/
    manage.py
    requirements.txt
  ```

2. Create text file called `environment.ini` with the following contents:

  ```
  [django]
  secret_key: <replace_with_your_secret>
  allowed_hosts: 127.0.0.1,localhost
  db_user: geo
  db_pass: geo
  ```

  Change the permissions to `0600` for security:
  ```
  chmod 0600 environment.ini
  ```

3. Add the file `geodjango/utils.py` with the following contents:

  This script will read `environment.ini`, which is .gitignored and contains private information not to be checked into version control.
  
  ```
  import os
  import string
  from filelock import FileLock
  from configparser import ConfigParser
  from django.core.exceptions import ImproperlyConfigured
  from django.utils.crypto import get_random_string
  
  VALID_KEY_CHARS = string.ascii_uppercase + string.ascii_lowercase + string.digits
  
  
  class FilePermissionError(Exception):
      """The file permissions are insecure."""
      pass
  
  
  def load_environment_file(envfile, key_length=64):
      config = None
      lock = FileLock(os.path.abspath(envfile) + ".lock")
      with lock:
          if not os.path.exists(envfile):
              # Create empty file if it doesn't exists
              old_umask = os.umask(0o177)  # Use '0600' file permissions
              config = ConfigParser()
              config.add_section('django')
              config['django']['secret_key'] = get_random_string(key_length, VALID_KEY_CHARS)
  
              with open(envfile, 'w') as configfile:
                  config.write(configfile)
              os.umask(old_umask)
  
          if (os.stat(envfile).st_mode & 0o777) != 0o600:
              raise FilePermissionError("Insecure environment file permissions for %s!" % envfile)
  
          if not config:
              config = ConfigParser()
              config.read_file(open(envfile))
  
          if not config.has_section('django'):
              raise ImproperlyConfigured('Missing `django` section in the environment file.')
  
          if not config.get('django', 'secret_key', fallback=None):
              raise ImproperlyConfigured('Missing `secret_key` in django section in the environment file.')
  
          # Register all keys as environment variables
          for key, value in config.items('django'):
              envname = 'DJANGO_%s' % key.upper()  # Prefix to avoid collisions with existing env variables
              if envname not in os.environ:  # Don't replace existing defined variables
                  os.environ[envname] = value
  ```

4. Alter `geodjango/settings.py`:
  This will take advantage of [django-configurations](https://github.com/jazzband/django-configurations) so that settings can be setup using Pythons class-based inheritance. This keeps 'Development' settings separate from 'Production' and/or 'Staging'.
  
  ```
  """
  Django settings for geodjango project.
  
  Generated by 'django-admin startproject' using Django 1.11.dev20160731183829.
  
  For more information on this file, see
  https://docs.djangoproject.com/en/dev/topics/settings/
  
  For the full list of settings and their values, see
  https://docs.djangoproject.com/en/dev/ref/settings/
  """
  
  import os
  from configurations import Configuration
  from .utils import load_environment_file
  load_environment_file('environment.ini')
  
  
  class Base(Configuration):
      # Build paths inside the project like this: os.path.join(BASE_DIR, ...)
      BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
  
      # Quick-start development settings - unsuitable for production
      # See https://docs.djangoproject.com/en/dev/howto/deployment/checklist/
  
      # SECURITY WARNING: keep the secret key used in production secret!
      SECRET_KEY = os.environ.get('DJANGO_SECRET_KEY', None)
  
      # Application definition
  
      INSTALLED_APPS = [
          'django.contrib.admin',
          'django.contrib.auth',
          'django.contrib.contenttypes',
          'django.contrib.sessions',
          'django.contrib.messages',
          'django.contrib.staticfiles',
          'django.contrib.gis', # GeoDjango
      ]
  
      MIDDLEWARE_CLASSES = [
          'django.middleware.security.SecurityMiddleware',
          'django.contrib.sessions.middleware.SessionMiddleware',
          'django.middleware.common.CommonMiddleware',
          'django.middleware.csrf.CsrfViewMiddleware',
          'django.contrib.auth.middleware.AuthenticationMiddleware',
          'django.contrib.messages.middleware.MessageMiddleware',
          'django.middleware.clickjacking.XFrameOptionsMiddleware',
      ]
  
      ROOT_URLCONF = 'geodjango.urls'
  
      TEMPLATES = [
          {
              'BACKEND': 'django.template.backends.django.DjangoTemplates',
              'DIRS': [],
              'APP_DIRS': True,
              'OPTIONS': {
                  'context_processors': [
                      'django.template.context_processors.debug',
                      'django.template.context_processors.request',
                      'django.contrib.auth.context_processors.auth',
                      'django.contrib.messages.context_processors.messages',
                  ],
              },
          },
      ]
  
      WSGI_APPLICATION = 'geodjango.wsgi.application'
  
      # Password validation
      # https://docs.djangoproject.com/en/dev/ref/settings/#auth-password-validators
  
      AUTH_PASSWORD_VALIDATORS = [
          {
              'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
          },
          {
              'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
          },
          {
              'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
          },
          {
              'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
          },
      ]
  
      # Internationalization
      # https://docs.djangoproject.com/en/dev/topics/i18n/
  
      LANGUAGE_CODE = 'en-us'
  
      TIME_ZONE = 'UTC'
  
      USE_I18N = True
  
      USE_L10N = True
  
      USE_TZ = True
  
  class Dev(Base):
      # SECURITY WARNING: don't run with debug turned on in production!
      DEBUG = True
  
      ALLOWED_HOSTS = os.environ.get('DJANGO_ALLOWED_HOSTS', '*').split()
  
      # Database
      # https://docs.djangoproject.com/en/dev/ref/settings/#databases
  
      DATABASES = {
          'default': {
              'ENGINE': 'django.contrib.gis.db.backends.postgis',
              'NAME': 'geodjango',
              'USER': os.environ.get('DJANGO_DB_USER', ''),
              'PASSWORD': os.environ.get('DJANGO_DB_PASS', ''),
          }
      }
  
      # Static files (CSS, JavaScript, Images)
      # https://docs.djangoproject.com/en/dev/howto/static-files/
  
      STATIC_URL = '/static/'
  ```

5. Alter `manage.py`:
  `django-configurations` must alter `manage.py` to use its execute_from_command_line method.
  
  ```
  #!/usr/bin/env python
  import os
  import sys
  
  if __name__ == "__main__":
      os.environ.setdefault("DJANGO_SETTINGS_MODULE", "geodjango.settings")
      os.environ.setdefault('DJANGO_CONFIGURATION', 'Dev')
      try:
          from configurations.management import execute_from_command_line
      except ImportError:
          # The above import may fail for some other reason. Ensure that the
          # issue is really that Django is missing to avoid masking other
          # exceptions on Python 2.
          try:
              import django
          except ImportError:
              raise ImportError(
                  "Couldn't import Django. Are you sure it's installed and "
                  "available on your PYTHONPATH environment variable? Did you "
                  "forget to activate a virtual environment?"
              )
          raise
      execute_from_command_line(sys.argv)
      
      ...
  ```

## Django application
The Django project has been bootstrapped and it ready for creating a sample geo-spatial application called `world`, which will provide all the world boundaries.

1. Create the world application

  ```
  python manage.py startapp world
  ```

2. Add to `geodjango/settings.py` `INSTALLED_APPS`:

  ```
  INSTALLED_APPS = [
      'django.contrib.admin',
      'django.contrib.auth',
      'django.contrib.contenttypes',
      'django.contrib.sessions',
      'django.contrib.messages',
      'django.contrib.staticfiles',
      'django.contrib.gis', # GeoDjango
      'world',
  ]
  ```

3. Get the geographic data

  This will download ESRI Shapefiles that represent the boarders of the world countries from [thematicmapping.org](http://thematicmapping.org)
  
  ```
  mkdir world/data
  cd world/data
  wget http://thematicmapping.org/downloads/TM_WORLD_BORDERS-0.3.zip
  unzip TM_WORLD_BORDERS-0.3.zip
  cd ../..
  ```

4. Create the WorldBorder model by editing `world/models.py`:

  ```
  from django.contrib.gis.db import models
  
  class WorldBorder(models.Model):
      # Regular Django fields corresponding to the attributes in the
      # world borders shapefile.
      name = models.CharField(max_length=50)
      area = models.IntegerField()
      pop2005 = models.IntegerField('Population 2005')
      fips = models.CharField('FIPS Code', max_length=2)
      iso2 = models.CharField('2 Digit ISO', max_length=2)
      iso3 = models.CharField('3 Digit ISO', max_length=3)
      un = models.IntegerField('United Nations Code')
      region = models.IntegerField('Region Code')
      subregion = models.IntegerField('Sub-Region Code')
      lon = models.FloatField()
      lat = models.FloatField()
  
      # GeoDjango-specific: a geometry field (MultiPolygonField)
      mpoly = models.MultiPolygonField()
  
      # Returns the string representation of the model.
      def __str__(self):              # __unicode__ on Python 2
          return self.name
  ```

5. Run migrate

	Django migration will create the database tables based on our model schema, as well as any of the models defined in `geodjango/settings` `INSTALLED_APPS`.

  ```
  python manage.py makemigrations
  python manage.py migrate
  ```

6. Create the `world/load.py` script to load the downloaded data:

  ```
  import os
  from django.contrib.gis.utils import LayerMapping
  from .models import WorldBorder
  
  world_mapping = {
      'fips' : 'FIPS',
      'iso2' : 'ISO2',
      'iso3' : 'ISO3',
      'un' : 'UN',
      'name' : 'NAME',
      'area' : 'AREA',
      'pop2005' : 'POP2005',
      'region' : 'REGION',
      'subregion' : 'SUBREGION',
      'lon' : 'LON',
      'lat' : 'LAT',
      'mpoly' : 'MULTIPOLYGON',
  }
  
  world_shp = os.path.abspath(
      os.path.join(os.path.dirname(__file__), 'data', 'TM_WORLD_BORDERS-0.3.shp'),
  )
  
  def run(verbose=True):
      lm = LayerMapping(
          WorldBorder, world_shp, world_mapping,
          transform=False, encoding='iso-8859-1',
      )
      lm.save(strict=True, verbose=verbose)
  ```

7. Launch Django shell and run:

  ```
  python manage.py shell
  ```
  ```
  from world import load
  load.run()
  ```

8. Enable the admin interface by creating `world/admin.py`:

  ```
  from django.contrib.gis import admin
  from .models import WorldBorder
  
  admin.site.register(WorldBorder, admin.OSMGeoAdmin)
  ```

9. Edit `geodjango/urls.py`:

  ```
  from django.conf.urls import url, include
  from django.contrib.gis import admin
  
  urlpatterns = [
      url(r'^admin/', admin.site.urls),
  ]
  ```

10. Create the Django admin user

  ```
  python manage.py createsuperuser
  ```

11. Start the development server

  ```
  python manage.py runserver --configurion=Dev
  ```

## Screenshot of Admin Interface

![Alt Text](https://github.com/dan-nyanko/geodjango/raw/master/static/images/admin-world-border-screenshot.jpeg)

## Further reading

The official Django tutorial provides [examples](https://docs.djangoproject.com/en/dev/ref/contrib/gis/tutorial/#spatial-queries) of filtering geo-spatial information stored in the database using the ORM.
