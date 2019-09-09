# django-restql
[![Build Status](https://api.travis-ci.com/yezyilomo/django-restql.svg?branch=master)](https://api.travis-ci.com/yezyilomo/django-restql)
[![Latest Version](https://img.shields.io/pypi/v/django-restql.svg)](https://pypi.org/project/django-restql/)
[![Python Versions](https://img.shields.io/pypi/pyversions/django-restql.svg)](https://pypi.org/project/django-restql/)
[![License](https://img.shields.io/pypi/l/django-restql.svg)](https://pypi.org/project/django-restql/)

**django-restql** is a python library which allows you to turn your API made with **Django REST Framework(DRF)** into a GraphQL like API. With this you will be able to
* Send a query to your API and get exactly what you need, nothing more and nothing less.

* Control the data you get, not the server.

* Get predictable results, since you control what you get from the server.

* Save the load of fetching unused data from the server(Over-fetching and Under-fetching problem).

Isn't it cool?.


## Installing

```python
pip install django-restql
```

## Querying Data
Using **django-restql** to query data is very simple, you just have to inherit the `DynamicFieldsMixin` class when defining a serializer.
```python
from rest_framework import serializers
from django.contrib.auth.models import User

from django_restql.mixins import DynamicFieldsMixin

class UserSerializer(DynamicFieldsMixin, serializer.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'groups']
```

A regular request returns all fields as specified on DRF serializer, in fact **django-restql** doesn't handle this request at all:

`GET /users`

``` json
    [
      {
        "id": 1,
        "username": "yezyilomo",
        "email": "yezileliilomo@hotmail.com",
        "groups": [1,2]
      },
      ...
    ]
```

**django-restql** handle all GET requests with `query` parameter, this parameter is the one used to pass all fields to be included in a response. For example to select `id` and `username` fields from `user` model, send a request with a ` query` parameter as shown below.

`GET /users/?query={id, username}`

```json
    [
      {
        "id": 1,
        "username": "yezyilomo"
      },
      ...
    ]
```

**django-restql** support querying both flat and nested resources, so you can expand or query nested fields at any level as long as your field is defined as nested field on a serializer. For example you can query a country and region field from location.

`GET /users/?query={id, username, location{country, region}}`

```json
    [
      {
        "id": 1,
        "username": "yezyilomo",
        "location": {
            "contry": "Tanzania",
            "region": "Dar es salaam"
        }
      },
      ...
    ]
```

**django-restql** got your back on querying iterable nested fields(one2many or many2many) too. For example if you want to expand `groups` field into `id` and `name`, here is how you would do it.

`GET /users/?query={id, username, groups{id, name}}`

```json
    [
      {
        "id": 1,
        "username": "yezyilomo",
        "groups": [
            {
                "id": 2,
                "name": "Auth_User"
            }
            {
                "id": 3,
                "name": "Admin_User"
            }
        ]
      },
      ...
    ]
```

If a query contains nested field without expanding and it's not defined as a nested field on a serializer, **django-restql** will return its id or array of ids for the case of nested iterable field(one2many or many2many). For example on a request below `location` is a flat nested field(many2one) and `groups` is an iterable nested field(one2many or many2many).

`GET /users/?query={id, username, location, group}`

```json
    [
      {
        "id": 1,
        "username": "yezyilomo",
        "location": 6,
        "groups": [1,2]
      },
      ...
    ]
```


## Using `fields=[..]` and `exclude=[..]` kwargs
With **django-restql** you can specify fields to be included when instantiating a serializer, this provides a way to refilter fields on nested fields(i.e you can opt to remove some fields on a nested field). Below is an example which shows how you can specify fields to be included on nested resources. 

```python
from rest_framework import serializers
from django.contrib.auth.models import User

from django_restql.mixins import DynamicFieldsMixin

class BookSerializer(DynamicFieldsMixin, serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'author']


class CourseSerializer(DynamicFieldsMixin, serializers.ModelSerializer):
    books = BookSerializer(many=True, read_only=True, fields=["title"])
    class Meta:
        model = Course
        fields = ['name', 'code', 'books']
```

`GET /courses/`

```json
    [
      {
        "name": "Computer Programming",
        "code": "CS50",
        "books": [
          {"title": "Computer Programming Basics"},
          {"title": "Data structures"}
        ]
      },
      ...
    ]
```
As you see from the response above, the nested resource(book) has only one field(title) as specified on  `fields=["title"]` kwarg during instantiating BookSerializer, so if you send a request like `GET /course?query={name, code, books{title, author}}` you will get an error that `author` field is not found because it was not included on `fields=["title"]` kwarg.


You can also specify fields to be excluded when instantiating a serializer by using `exclude=[]` as shown below 
```python
from rest_framework import serializers
from django.contrib.auth.models import Book, Course

from django_restql.mixins import DynamicFieldsMixin

class BookSerializer(DynamicFieldsMixin, serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'author']


class CourseSerializer(DynamicFieldsMixin, serializers.ModelSerializer):
    books = BookSerializer(many=True, read_only=True, exclude=["author"])
    class Meta:
        model = Course
        fields = ['name', 'code', 'books']
```

`GET /courses/`

```json
    [
      {
        "name": "Computer Programming",
        "code": "CS50",
        "books": [
          {"id": 1, "title": "Computer Programming Basics"},
          {"id": 2, "title": "Data structures"}
        ]
      },
      ...
    ]
```
From the response above you can see that `author` field has been excluded fom book nested resource as specified on  `exclude=["author"]` kwarg during instantiating BookSerializer.

**Note**: `fields=[..]` and `exclude=[]` kwargs have no effect when you access the resources directly, so when you access books you will still get all fields i.e

`GET /books/`

```json
    [
      {
        "id": 1,
        "title": "Computer Programming Basics",
        "author": "S.Mobit"
      },
      ...
    ]
```
So you can see that all fields have appeared as specified on `fields = ['id', 'title', 'author']` on BookSerializer class.

## Using `return_pk=True` kwargs
With **django-restql** you can specify whether to return nested resource pk or data. Below is an example which shows how we can specify fields to be included on nested resources. 

```python
from rest_framework import serializers
from django.contrib.auth.models import Book, Course

from django_restql.mixins import DynamicFieldsMixin

class BookSerializer(DynamicFieldsMixin, serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'author']


class CourseSerializer(DynamicFieldsMixin, serializers.ModelSerializer):
    books = BookSerializer(many=True, read_only=True, return_pk=True)
    class Meta:
        model = Course
        fields = ['name', 'code', 'books']
```

`GET /course/`

```json
    [
      {
        "name": "Computer Programming",
        "code": "CS50",
        "books": [1,2]
      },
      ...
    ]
```
So you can see that on a nested field `books` book pks have been returned instead of books data as specified on `return_pk=True` kwarg on `BookSerializer`.


## Customizing django-restql
**django-restql**  is very configurable, here is what you can customize on it.
* Change the name of ```query``` parameter.

    If you don't want to use the name ```query``` as your parameter, you can inherit `DynamicFieldsMixin` and change it as shown below
    ```python
    from django_restql.mixins import DynamicFieldsMixin

    class MyDynamicFieldMixin(DynamicFieldsMixin):
        query_param_name = "your_favourite_name"
     ```
     Now you can use this Mixin on your serializer and use the name `your_favourite_name` as your parameter. E.g

     `GET /users/?your_favourite_name={id, username}`

* Customize how fields to include in a response are filtered.
    You can do this by inheriting DynamicFieldsMixin and override `field` methods as shown below.

    ```python
    from django_restql.mixins import DynamicFieldsMixin

    class CustomDynamicFieldMixin(DynamicFieldsMixin):
        @property
        def fields(self):
            # Your customization here
            return fields
    ```
    **Note:** To be able to do this you must understand how **django-restql** is implemented, specifically **DynamicFieldsMixin** class, you can check it [here](https://github.com/yezyilomo/django-restql/blob/master/django_restql/mixins.py). In fact this is how **django-restql** is implemented(just by overriding `field` method of a serializer, nothing more and nothing less).


## Mutating Data(Creating and Updating Data)
**django-restql** got your back on creating and updating nested data too, it has two components for mutating nested data, `NestedModelSerializer` and `NestedField`. A serializer `NestedModelSerializer` has `update` and `create` logics for nested fields on the other hand `NestedField` is used to validate data before dispatching update or create.


### Using NestedField & NestedModelSerializer to mutate data
Just like in querying data, mutating nested data with **django-restql** is very simple, you just have to inherit `NestedModelSerializer` on a serializer with nested fields and use `NestedField` to define those nested fields. Below is an example which shows how to use `NestedModelSerializer` and `NestedField`.
```python
from app.models import Location, Amenity, Property
from django_restql.serializers import NestedModelSerializer
from django_restql.fields import NestedField


class LocationSerializer(NestedModelSerializer):
    class Meta:
        model = Location
        fields = ("id", "city", "country")


class AmenitySerializer(NestedModelSerializer):
    class Meta:
        model = Amenity
        fields = ("id", "name")
        

class PropertySerializer(NestedModelSerializer):
    location = NestedField(LocationSerializer)
    amenities = NestedField(AmenitySerializer, many=True)
    class Meta:
        model = Property
        fields = (
            'id', 'price', 'location', 'amenities'
        )
```
<br>


```POST /api/property/```

Request Body
```json
{
    "price": 60000,
    "location": {
        "city": "Newyork",
        "country": "USA"
    },
    "amenities": {
        "add": [3],
        "create": [
            {"name": "Watererr"},
            {"name": "Electricity"}
        ]
    }
}
```
What's done here is pretty clear, location will be created and associated with the property created, also create operation on amenities will create amenities with values specified in a list and associate with the property, add operation will add amenity with id 4 to a list of amenities of the property.

**Note**: POST for many related field supports two operations which are `create` and `add`.

<br>

Response
```json
{
    "id": 2,
    "price": 60000,
    "location": {
        "id": 3,
        "city": "Newyork",
        "country": "USA"
    },
    "amenities": [
        {"id": 1, "name": "Watererr"},
        {"id": 2, "name": "Electricity"},
        {"id": 3, "name": "Swimming Pool"}
    ]
}
```
<br>


```PUT /api/property/2/```

Request Body
```json
{
    "price": 50000,
    "location": {
        "city": "Newyork",
        "country": "USA"
    },
    "amenities": {
        "add": [4],
        "create": [{"name": "Fance"}],
        "remove": [3],
        "update": {1: {"name": "Water"}}
    }
}
```
**Note**: Here `add`, `create`, `remove` and `update` are operations, so `add` operation add amenitiy with id 4 to a list of amenities of the property, `create` operation create amenities with values specified in a list, `remove` operation dessociate amenities with id 3 from a property, `update` operation edit amenity with id 1 according to values specified.

**Note**: PUT/PATCH for many related field supports four operations which are `create`, `add`, `remove` and `update`.

<br>

Response
```json
{
    "id": 2,
    "price": 50000,
    "location": {
        "id": 3,
        "city": "Newyork",
        "country": "USA"
    },
    "amenities": [
        {"id": 1, "name": "Water"},
        {"id": 2, "name": "Electricity"},
        {"id": 4, "name": "Bathtub"},
        {"id": 5, "name": "Fance"}
    ]
}
```
<br>
<br>


### Using NestedField with `accept_pk=True` kwarg.
`accept_pk=True` is used if you want to update nested field by using pk/id of existing data(basically associate and dessociate existing nested resources with the parent resource without actually mutating the nested resource). This applies to ForeignKey relation only.

```python
from app.models import Location, Amenity, Property
from django_restql.serializers import NestedModelSerializer 
from django_restql.fields import NestedField


class LocationSerializer(NestedModelSerializer):
    class Meta:
        model = Location
        fields = ("id", "city", "country")


class PropertySerializer(NestedModelSerializer):
    location = NestedField(ocationSerializer, accept_pk=True)
    class Meta:
        model = Property
        fields = (
            'id', 'price', 'location'
        )
```
<br>


```POST /api/property/```

Request Body
```json
{
    "price": 40000,
    "location": 2
}
```
**Note**: Here location resource with id 2 is already existing, so what's done here is create new property resource and associate it with a location with id 2.
<br>

Response
```json
{
    "id": 1,
    "price": 40000,
    "location": {
        "id": 2,
        "city": "Tokyo",
        "country": "China"
    }
}
```
<br>


### Using NestedField with `create_ops=[..]` and `update_ops=[..]` kwargs.
You can restrict some operations by using `create_ops` and `update_ops` keyword arguments as follows

```python
from app.models import Location, Amenity, Property
from django_restql.serializers import NestedModelSerializer 
from django_restql.fields import NestedField


class AmenitySerializer(NestedModelSerializer):
    class Meta:
        model = Amenity
        fields = ("id", "name")
        

class PropertySerializer(NestedModelSerializer):
    amenities = NestedField(
        AmenitySerializer, 
        many=True,
        create_ops=["add"],  # Allow only add operation(restrict create operation)
        update_ops=["add", "remove"]  # Allow only add and remove operations(restrict create and update operations)
    )
    class Meta:
        model = Property
        fields = (
            'id', 'price', 'amenities'
        )
```
<br>


```POST /api/property/```

Request Body
```json
{
    "price": 60000,
    "amenities": {
        "add": [1, 2]
    }
}
```
**Note**: According to `create_ops=["add"]`, you can't use `create` operation in here!.
<br>

Response
```json
{
    "id": 2,
    "price": 60000,
    "amenities": [
        {"id": 1, "name": "Watererr"},
        {"id": 2, "name": "Electricity"}
    ]
}
```
<br>


```PUT /api/property/2/```

Request Body
```json
{
    "price": 50000,
    "amenities": {
        "add": [3],
        "remove": [2]
    }
}
```
**Note**: According to `update_ops=["add", "remove"]`, you can't use `create` or `update` operation in here!.
<br>

Response
```json
{
    "id": 2,
    "price": 50000,
    "amenities": [
        {"id": 1, "name": "Water"},
        {"id": 3, "name": "Bathtub"}
    ]
}
```
<br>

## Running Tests
`python setup.py test`


## Credits
* Implementation of this library is based on the idea behind [GraphQL](https://graphql.org/).
* My intention is to extend the capability of [drf-dynamic-fields](https://github.com/dbrgn/drf-dynamic-fields) library to support more functionalities like allowing to query nested fields both flat and iterable at any level and allow writing on nested fields while maintaining simplicity.


## Contributing [![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](http://makeapullrequest.com)

We welcome all contributions. Please read our [CONTRIBUTING.md](https://github.com/yezyilomo/django-restql/blob/master/CONTRIBUTING.md) first. You can submit any ideas as [pull requests](https://github.com/yezyilomo/django-restql/pulls) or as [GitHub issues](https://github.com/yezyilomo/django-restql/issues). If you'd like to improve code, check out the [Code Style Guide](https://github.com/yezyilomo/django-restql/blob/master/CONTRIBUTING.md#styleguides) and have a good time!.
