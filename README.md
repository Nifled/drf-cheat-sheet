# drf-cheat-sheet
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
2. [Serialization](#Serialization)
    - [ModelSerializer](#using-modelserializer-class)
    - [Nested Serialization](#nested-serialization)
    - [HyperlinkedModelSerializer](#hyperlinkedmodelserializer)
3. [Views](#views)
    - [Function-based views](#using-function-based-views)
    - [Class-based views](#using-class-based-views)
    - [Generic Class-based views](#using-generic-class-based-views)
    - [Mixins](#using-mixins)
    - [ViewSets](#using-viewsets)
        * [Routers](#routers)
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


### Serialization

Serializers allow complex data like querysets and model instances to be converted to native Python datatypes that can then be easily rendered into JSON, XML, and other formats.

#### Using ModelSerializer class:

Suppose we wanted to create a PostSerializer for our example [Post](#base-example-model) model and CommentSerializer for our [Comment](#base-example-model) model.

```python
class PostSerializer(serializers.ModelSerializer):

    class Meta:
        model = Post
        fields = ('id', 'title', 'text', 'created')
        
        
class CommentSerializer(serializers.ModelSerializer):

    class Meta:
        model = Comment
        fields = ('post', 'user', 'text')
```
Or also you could use `exclude` to exclude certain fields from being seialized. ModelSerializer has default implementations for the `create()` and `update()` methods.

##### Nested Serialization

By default, instances are serialized with primary keys to represent relationships. To get nested serialization we could use, *General* or *Explicit* methods.

###### General

Using `depth` parameter.

```python
class CommentSerializer(serializers.ModelSerializer):

    class Meta:
        model = Comment
        fields = '__all__'
        depth = 2
```

###### Explicit

When you don't want all the nested fields to be serialized...

```python
class CommentSerializer(serializers.ModelSerializer):
    post = PostSerializer()
    
    class Meta:
        model = Comment
        fields = '__all__'
```

#### HyperlinkedModelSerializer

This makes your web API a lot more easy to use (in browser) and would be a nice feature to add.

Lets say we need to see the comments every post has in each of the [Post](#base-example-model) instances of our API and each [Comment](#base-example-model) needs to link(with url) to it's `detail` view.

```python
class PostSerializer(serializers.HyperlinkedModelSerializer):

    class Meta:
        model = Post
        fields = ('id', 'title', 'text', 'created', 'comments')
        read_only_fields = ('comments',)
```

**Note:** without the `read_only_fields`, the `create` form for Posts would always require a `comments` input, which doesn't make sense (comments on a post are normally made AFTER the post is created).

Another way of hyperlinking is just adding a `HyperlinkedRelatedField` definition to a normal serializer.

```python
class PostSerializer(serializers.ModelSerializer):
    comments = serializers.HyperlinkedRelatedField(many=True, view_name='comment-detail', read_only=True)
    
    class Meta:
        model = Post
        fields = ('id', 'title', 'text', 'created', 'comments')
```




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
from rest_framework import generics, mixins
from posts.models import Post
from posts.serializers import PostSerializer


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

#### Using ViewSets:

With `ModelViewSet` (in this case), you don't have to create separate views for getting list of objects and detail of one object. ViewSet will handle it for you in a consistent way for both methods.

```python
from rest_framework import viewsets
from posts.models import Post
from posts.serializers import PostSerializer


class PostViewSet(viewsets.ModelViewSet):
    """
    A viewset for viewing and editing post instances.
    """
    queryset = Post.objects.all()
    serializer_class = PostSerializer
```

So basically, this would not only generate the `list` view, but also the `detail` view for every [Post](#base-example-model) instance.

##### Routers

Routers in ViewSets allow the URL configuration for your API to be automatically generated using naming standards.

```python
from rest_framework.routers import DefaultRouter
from posts.views import PostViewSet

router = DefaultRouter()
router.register(r'users', UserViewSet)
urlpatterns = router.urls
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

OAuth and OAuth2 were previously integrated in DRF, but the corresponding modules were moved and is now supported as a third-party package. There are also other very cool and handy packages that can be easily implemented.

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
