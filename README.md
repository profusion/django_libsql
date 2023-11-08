# django_libsql

libSQL backend compatibility for django.

# Pre-requisites

You must have a running instance of [sqld](https://github.com/libsql/sqld), which is the libSQL server mode. There are several supported options:

- [Build and run an instance](https://github.com/libsql/sqld/blob/main/docs/BUILD-RUN.md) on your local machine.
- Use an instance managed by [Turso](https://turso.tech/).
- Use the [libSQL test server](https://github.com/libsql/hrana-test-server) implemented in python

# Co-requisites

This custom backend requires django version 4.1.0 or higher and [libsql_client](https://pypi.org/project/libsql-client/)

# Getting started

In your django project settings, if you're using the sqlite engine you'll find something like this

``` python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
```

Just change the engine name to `django_libsql` and your database name to the appropriate Turso URL

``` python
DATABASES = {
    'default': {
        'ENGINE': 'django_libsql',
        'NAME': 'libsql://you-database-hostname?authToken=your-auth-token'
    }
}
```

Or, if you're running a local instance of sqld, the name will be just `libsql://127.0.0.1:PORT`.

# Known issues

The implementation of the standard sqlite backend for django uses the
[create_function api](https://docs.python.org/3/library/sqlite3.html#sqlite3.Connection.create_function)
in order to define some custom functionalities of the framework.
We currently do not support this feature in the [libsql_client](https://pypi.org/project/libsql-client/), which means we don't support the functions
listed [here](https://github.com/django/django/blob/8b1acc0440418ac8f45ba48e2dfcf5126c83341b/django/db/backends/sqlite3/_functions.py#L39-L102).

You can see which functions are registered to a given database by querying the `pragma_function_list` table. When running django with the standard
sqlite backend, the functions mentioned above will be registered, whereas with `django_libsql`, they won't.

More concretely, here are some of the things that will not work in the current version:

Let the model `Product` be defined by:

``` python
    class Product(models.Model):
        made_at = models.DateTimeField()
        valid_until = models.DateTimeField()
```

Then

``` python
    >>> from django.db.models import F
    >>> Product.objects.annotate(date_diff=F("valid_until") - F("made_at"))
```
will raise:
`django.db.utils.OperationalError: SQL_INPUT_ERROR: SQL input error: no such function: django_timestamp_diff (at offset 87)`

Let the model `ShcoolClass` be defined by:

``` python
    class SchoolClass(models.Model):
        year = models.PositiveIntegerField()
        day = models.CharField(max_length=9, blank=True)
        last_updated = models.DateTimeField()

        objects = SchoolClassManager()
```

Then

``` python
    >>> import datetime
    >>> updated = datetime.datetime(2023, 2, 20)
    >>> SchoolClass.objects.create(year=2022, last_updated=updated)
    <SchoolClass: SchoolClass object (3)>
    >>> years = SchoolClass.objects.dates("last_updated", "year")
    >>> years
```

will raise:
`django.db.utils.OperationalError: SQL_INPUT_ERROR: SQL input error: no such function: django_date_trunc (at offset 16)`

These are some examples, but in general, any framework feature that uses the functions
listed
[here](https://github.com/django/django/blob/8b1acc0440418ac8f45ba48e2dfcf5126c83341b/django/db/backends/sqlite3/_functions.py#L39-L102)
will not work for the time being.


# License

django_libsql is distributed under the [MIT license](https://opensource.org/licenses/MIT).