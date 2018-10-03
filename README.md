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

2. [Serialization](#serialization)
    - [ModelSerializer](#using-modelserializer-class)
    - [Nested Serialization](#nested-serialization)
    - [HyperlinkedModelSerializer](#hyperlinkedmodelserializer)
    - [Dynamically modifying fields in serializer](#dynamically-modifying-fields-in-serializer)

3. [Views](#views)
    - [Function-based views](#using-function-based-views)
    - [Class-based views](#using-class-based-views)
    - [Generic Class-based views](#using-generic-class-based-views)
    - [Mixins](#using-mixins)
    - [ViewSets](#using-viewsets)
        * [Routers](#routers)
        * [Custom actions](#custom-actions=in-viewsets)

4. [Pagination](#pagination)
    - [With Generic Class-based views or Viewsets](#with-generic-class-based-views-or-viewsets)
    - [With APIView or other Non-Generic View](#with-apiview-or-other-non-generic-view)
    - [Modify Pagination Class](#modify-pagination-class)

5. [Authentication](#authentication)
    - [SessionAuthentication](#sessionauthentication)
    - [TokenAuthentication](#tokenauthentication)
        * [Generating Tokens](#generate-tokens-for-users)
        * [Obtaining Tokens](#obtaining-tokens)
    - [OAuth2](#oauth2)

6. [Web Browsable API](#web-browsable-api)
    - [Overriding the Default Theme](#overriding-the-default-theme)
    - [Full Customization](#full-customization)

7. [Testing](#testing)
    - [APIRequestFactory](#apirequestfactory)


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

<br/>

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

Yuo can also define and nest serializers within eachother...

```python
class CommentSerializer(serializers.ModelSerializer):
    post = PostSerializer()
    
    class Meta:
        model = Comment
        fields = '__all__'
```

So here, the comment's `post` field (how we named it in models.py) will serialize however we defined it in `PostSerializer`.

#### HyperlinkedModelSerializer

This makes your web API a lot more easy to use (in browser) and would be a nice feature to add.

Lets say we wanted to see the comments that every post has in each of the [Post](#base-example-model) instances of our API. With `HyperlinkedModelSerializer`, instead of having nested primary keys or nested fields, we get a link to each individual [Comment](#base-example-model) (url).

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

#### Dynamically modifying fields in serializer

This makes your web API a lot more easy for extract limited number of parameter in response. Let's say you want to set which fields should be used by a serializer at the point of initialization.

Just copy below code and past it in your serliazer file
```python
class DynamicFieldsModelSerializer(serializers.ModelSerializer):

    def __init__(self, *args, **kwargs):
        # Don't pass the 'fields' arg up to the superclass
        fields = kwargs.pop('fields', None)

        # Instantiate the superclass normally
        super(DynamicFieldsModelSerializer, self).__init__(*args, **kwargs)

        if fields is not None:
            # Drop any fields that are not specified in the `fields` argument.
            allowed = set(fields)
            existing = set(self.fields.keys())
            for field_name in existing - allowed:
                self.fields.pop(field_name)
```

Extend ```DynamicFieldsModelSerializer``` from your serializer class
```python
class UserSerializer(DynamicFieldsModelSerializer):
    class Meta:
        model = User
        fields = ('id', 'username', 'email')
```

Mention the fields name inside ```fields```
```python
UserSerializer(user, fields=('id', 'email'))
```

Here, you will get only ```id```and ```email``` from serializer instead of all.

[Back to Top ↑](#drf-cheat-sheet)

<br/>

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

##### Custom Actions in ViewSets
DRF provides helpers to add custom actions for _ad-hoc_ behaviours with the `@action` decorator. The router will configure its url accordingly.
For example, we can add a `comments` action in the our `PostViewSet` to retrieve all the comments of a specific post as follows:

 ```python
from rest_framework import viewsets
from rest_framework.decorators import action
from posts.models import Post
from posts.serializers import PostSerializer, CommentSerializer


class PostViewSet(viewsets.ModelViewSet):
    ...
    
    @action(methods=['get'], detail=True)
    def comments(self, request, pk=None):
        try:
            post = Post.objects.get(id=pk)
        except Post.DoesNotExist:
            return Response({"error": "Post not found."},
                            status=status.HTTP_400_BAD_REQUEST)
        comments = post.comments.all()
        return Response(CommentSerializer(comments, many=True))
```

Upon registering the view as `router.register(r'posts', PostViewSet)`, this action will then be available at the url `posts/{pk}/comments/`.

[Back to Top ↑](#drf-cheat-sheet)

<br/>

### Pagination

This can usually be performed automatically with `generic class-based views` and `viewsets`. If you're using `APIView`, it must be explicitly applied.

#### With Generic Class-based views or Viewsets

For these views to use pagination, all we have to do is override `DEFAULT_PAGINATION_CLASS` and `PAGE_SIZE` in our DRF settings in `settings.py`.

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination',
    'PAGE_SIZE': 20
}
```

#### With APIView or other Non-Generic View

First we must define the default behaviour for our pagination, which is done in `settings.py`.

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination',
    'PAGE_SIZE': 20
}
```

Now we need to handle our pagination through our view (APIView in this case).

```python
from rest_framework.settings import api_settings
from rest_framework.views import APIView

class PostView(APIView):
    # ... queryset, serializer_class, etc
    pagination_class = api_settings.DEFAULT_PAGINATION_CLASS

    # We need to override the get method to achieve pagination
    def get(self, request):
        # ...
        page = self.paginate_queryset(self.queryset)
        if page is not None:
            serializer = self.serializer_class(page, many=True)
            return self.get_paginated_response(serializer.data)
        # ...

    # Now add the pagination handlers taken from 
    # django-rest-framework/rest_framework/generics.py
    @property
    def paginator(self):
        """
        The paginator instance associated with the view, or `None`.
        """
     if not hasattr(self, '_paginator'):
         if self.pagination_class is None:
             self._paginator = None
         else:
             self._paginator = self.pagination_class()
     return self._paginator

     def paginate_queryset(self, queryset):
         """
         Return a single page of results, or `None` if pagination is disabled.
         """
         if self.paginator is None:
             return None
         return self.paginator.paginate_queryset(queryset, self.request, view=self)

     def get_paginated_response(self, data):
         """
         Return a paginated style `Response` object for the given output data.
         """
         return self.paginator.get_paginated_response(data)
```

#### Modify Pagination Class

We can define our own pagination class and override default attributes. Using this new class, you can make one, multiple or all views use this instead of `LimitOffsetPagination` or other ones provided by DRF.

```python
class CustomPagination(PageNumberPagination):
    """
    Client controls the response page size (with query param), limited to a maximum of `max_page_size` and default of `page_size`.
    """
    page_size = 100
    page_size_query_param = 'page_size'
    max_page_size = 10000
```

[Back to Top ↑](#drf-cheat-sheet)

<br/>

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

[Back to Top ↑](#drf-cheat-sheet)

<br/>

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

[Back to Top ↑](#drf-cheat-sheet)

<br/>

### Testing

It's important to test your API to make sure it works, of course.

#### APIRequestFactory

It has a similar name to Django's [RequestFactory](https://docs.djangoproject.com/en/1.11/topics/testing/advanced/#the-request-factory) class, because it extends it. You can use the different class methods to send test requests to your API.

Lets say we want to write test requests for our example [Post](#base-example-model) model.

```python
from django.test import TestCase
from rest_framework.test import APIRequestFactory
from posts.models import Post
from posts.views import PostList


class PostTest(TestCase):  # Post object, not HTTP method POST.
    """We'll be testing with the PostList view (class-based view)."""

    def setUp(self):
        self.factory = APIRequestFactory()
        self.post = Post.objects.create(title='Post example', text='Lorem Ipsum')
        
    # For HTTP method GET
    def get(self):
        view = PostList.as_view()
        request = self.factory.get('/posts/')
        response = view(request)
        
        self.assertEqual(response.status_code, 200)  # 200 = OK

    # For HTTP method POST
    def post(self):
        view = PostList.as_view()
        
        # Generating the request
        request = self.factory.post('/posts/', {'title': 'Post example', 'text': 'Lorem Ipsum'})
        
        response = view(request)
        expected = {'title': self.post.title, 'text': self.post.text}
        
        self.assertEqual(response.status_code, 201)  # 201 = created
        self.assertEqual(response.data, expected)

```

[Back to Top ↑](#drf-cheat-sheet)

<br/>

