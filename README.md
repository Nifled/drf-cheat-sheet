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
    - [ModelSerializer](#using-modelserializer-class)
3. [Views](#views)
    - [Function-based views](#using-function-based-views)
    - [Class-based views](#using-class-based-views)
    - [Generic Class-based views](#using-generic-class-based-views)
    - [Mixins](#using-mixins)
    


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

We'll write a `post_list` view with `GET` and `POST` methods for reading and creating new [Post](#base-example-model) instances.

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
```python
from rest_framework.response import Response
from rest_framework import status
from rest_framework.views import APIView
from posts.models import Post
from posts.serializers import PostSerializer


class PostList(APIView):
    def get(self, request, format=None):
        snippets = Post.objects.all()
        serializer = PostSerializer(snippets, many=True)
        return Response(serializer.data)

    def post(self, request, format=None):
        serializer = PostSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)

        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```


#### Using Generic Class-based views:

```python
from rest_framework import generics
from posts.models import Post
from posts.serializers import PostSerializer


class PostList(generics.ListCreateAPIView):
    queryset = Post.objects.all()
    serializer_class = PostSerializer
```

#### Using Mixins:

```python
from posts.models import Post
from posts.serializers import PostSerializer
from rest_framework import generics, mixins


class PostList(generics.GenericAPIView,
               mixins.ListModelMixin,
               mixins.CreateModelMixin
               ):
    queryset = Post.objects.all()
    serializer_class = PostSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```
