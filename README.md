# drf-cheat-sheat
A collection of anything from basics to advanced methods and usages with Django REST Framework for creating browsable and awesome web API's.

Here is DRF's [official documentation](http://www.django-rest-framework.org/) in case you need everything in detail.

### Why Django REST Framework?
Summarized from the official docs:

* Web browsable API
* Serialization that supports ORM and non-ORM data sources.
* Authentication is easily implemented.
* Highly customizable in every sense of the word.

### Installation

Install the package via `pip`:
```
$ pip install djangorestframework
```

Then, add `'rest_framework'` to `INSTALLED_APPS` in your `settings.py` file.

```
INSTALLED_APPS = [
    # Rest of your installed apps ...
    'rest_framework',
]
```
