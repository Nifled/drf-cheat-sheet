# drf-cheat-sheat
A collection of anything from basics to advanced recommended methods and usages with Django REST Framework for creating browsable and awesome web API's. This could also serve as a quick reference guide.

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
4. [Authentication](#authentication)
    - [SessionAuthentication](#sessionauthentication)
    - [TokenAuthentication](#tokenauthentication)
        * [Generating Tokens](#generate-tokens-for-users)
        * [Obtaining Tokens](#obtaining-tokens)
    - [OAuth2](#oauth2)
5. [Web Browsable API](#web-browsable-api)
    - [Overriding the Default Theme](#overriding-the-default-theme)
    - [Full Customization](#full-customization)


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

### Authentication

DRF has ready-to-use and integrated authentication schemes, but if you need something more specific, you can customize [your own scheme](http://www.django-rest-framework.org/api-guide/authentication/#custom-authentication).

#### SessionAuthentication

SessionAuthentication uses the default session backend for authentication provided by Django, which is more practical for us devs. Once a user has been successfully authenticated, a User instance is stored in `request.user`.


#### TokenAuthentication

The recommended use for the`TokenAuthentication` class is when client-server setups like native apps.

First, add `'rest_framework.authtoken'` to your `INSTALLED_APPS`:
```python
INSTALLED_APPS = [
    # Rest of your installed apps ...
    'rest_framework',
    'rest_framework.authtoken'
]
```

##### Generate Tokens for users
 
Using signals (in `models.py`).

```python
from django.conf import settings
from django.db.models.signals import post_save
from django.dispatch import receiver
from rest_framework.authtoken.models import Token

# For existing users
for user in User.objects.all():
    Token.objects.get_or_create(user=user)

# For newly created users
@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_auth_token(sender, instance=None, created=False, **kwargs):
    if created:
        Token.objects.create(user=instance)
```

##### Obtaining Tokens
DRF provides a built in view to obtain tokens given username and password.

```python
from rest_framework.authtoken import views
urlpatterns += [
    url(r'^api-token-auth/', views.obtain_auth_token)
]
```

#### OAuth2

OAuth and OAuth2 were previously integrated in DRF, but the corresponding modules were were moved and is now supported as a third-party package. There are also other very cool and handy packages that can be easily implemented.

* [Django Rest Framework OAuth](http://jpadilla.github.io/django-rest-framework-oauth/)
* [Django OAuth Toolkit](https://github.com/evonove/django-oauth-toolkit) (recommended for OAuth2)

If that isn't enough, there's a few more [here](http://www.django-rest-framework.org/topics/third-party-packages/#authentication).


### Web Browsable API

The default look DRF gives you for the browsable API is pretty cool on its own, but in case you don't like it, there are provided ways of customization.

#### Overriding the Default theme

First thing you must do is create a template in `templates/rest_framework/api.html` that extends `rest_framework/base.html`.

```
{% extends "rest_framework/base.html" %}
```

Now you can modify the many block components that are included in the `base.html` to your styling. Just as you would do on normal Django templates when you have a `base.html`.

* `body` - Whole HTML body.
* `bootstrap_theme` - CSS for the Bootstrap theme.
* `bootstrap_navbar_variant` - CSS for only the Navbar.
* `branding` - Brand component in Navbar (top left).
* `script` - Custom Javascript.
* `style` - Custom CSS.

Those are the common ones, here's [all of 'em](http://www.django-rest-framework.org/topics/browsable-api/#blocks).

```html
{% block branding %}
    <a class="navbar-brand" rel="nofollow" href="erickdelfin.me">
        Nifled's Blog
    </a>
{% endblock %}
```

This is how you would modify any other block.

#### Full Customization

If you don't dig the Bootstrap look, you can just drop the whole default look and fully customize it on your own. There is a context provided that you could work with.

* `api_settings` -  API settings
* `content` - The content of the API response
* `request` - The request object
* `response ` - The response object
* `view ` - The view handling the request

Full list of context variables [here](http://www.django-rest-framework.org/topics/browsable-api/#context).

Take a look at the actual [base HTML](https://github.com/encode/django-rest-framework/blob/73ad88eaae2f49bfd09508f2dcd6446677800a26/rest_framework/templates/rest_framework/base.html) source code for the API in DRF to get an idea of how it's actually made.
