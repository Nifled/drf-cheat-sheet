# drf-cheat-sheat
A collection of anything from basics to advanced recommended methods and usages with Django REST Framework for creating browsable and awesome web API's.

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
3. [Views](#views)
    - [Function-based views](#using-function-based-views)


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

```python
INSTALLED_APPS = [
    # Rest of your installed apps ...
    'rest_framework',
]
```


### Serializers

Serializers allow complex data like querysets and model instances to be converted to native Python datatypes that can then be easily rendered into JSON, XML, and other formats.

#### Using ModelSerializer class:

Suppose we wanted to create a PostSerializer for our example [Post](#base-example-model) model.

```python
class PostSerializer(serializers.ModelSerializer):
    class Meta:
        model = Post
        fields = ('id', 'title', 'text', 'created')
```

ModelSerializer has default implementations for the `create()` and `update()` methods.


### Views

There are many options for creating views for your web API, it really depends on what you want and personal preference.

We'll write a `post_list` view with `GET` and `POST` methods for reading and creating new post instances.

#### Using Function-based views:

```python
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework.parsers import JSONParser
from rest_framework import status
from posts.models import Post
from posts.serializers import PostSerializer

@api_view(['GET', 'POST'])
def post_list(request, format=None):

    if request.method == 'GET':
        posts = Post.objects.all()
        serializer = PostSerializer(posts, many=True)
        return Response(serializer.data)

    elif request.method == 'POST':
        data = JSONParser().parse(request)
        serializer = PostSerializer(data=data)

        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

#### Using Class-based views:
#### Using Generic class-based views:
#### Using Mixins:
