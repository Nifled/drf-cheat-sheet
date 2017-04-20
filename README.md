# drf-cheat-sheat
A collection of anything from basics to advanced methods and usages with Django REST Framework for creating browsable and awesome web API's.

Here is DRF's [official documentation](http://www.django-rest-framework.org/) in case you need everything in detail.

### Why Django REST Framework?
Summarized from the official docs:

* Web browsable API
* Serialization that supports ORM and non-ORM data sources.
* Authentication is easily implemented.
* Highly customizable in every sense of the word.

## Index

1. [Installation](#installation)
2. [Serializers](#serializers)


### Base Example Model

Throughout this cheat-sheet, the examples used will be based on this Post and Comment model.

```python
class Post(models.Model):  
    title = models.CharField(max_length=1024)
    text = models.TextField()
    created = models.DateField(auto_now_add=True)


class Comment(models.Model):
    post = models.ForeignKey(Post, related_name='comments')
    user = models.ForeignKey(User)
    text = models.TextField()
```

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

### Serializers

Serializers allow complex data such as querysets and model instances to be converted to native Python datatypes that can then be easily rendered into JSON, XML, and other formats.

##### Using ModelSerializer class:

Suppose we wanted to create a PostSerializer for our example [Post](#base-example-model) model.

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ('id', 'title', 'text', 'created')
```

Serializers and ModelSerializers are similar to Forms and ModelForms. Unlike forms, they are not constrained to dealing with HTML output, and form encoded input.

