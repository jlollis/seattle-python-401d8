# Django Relationships

In this lecture we’ll learn a bit about the Django database ORM, how we interact with it, and how it allows us to deal with relationships between objects.

* * *

## Users

We’ll start Django today using the `shell` management command:
```bash
$ python manage.py shell
```
What this means we can import any of the apps we have set up as being installed in our project.

_settings.py_
```python
# Application definition

INSTALLED_APPS = (
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'django.contrib.gis',
    'patron_profile',
)
```
This means in our Django shell we can import our profile model and the Django `User` model to play with them:
```
In [1]: from patron_profile.models import PatronProfile
In [2]: from django.contrib.auth.models import User
```
How do we query to see if we have any users in our DB? Remember, Django prefers to use the model Class object as the source for database interactions. Any model we create (or that is provided for us by Django) will have an `objects` attribute. That attribute is provided for us for free by Django. The value of the attribute is an instance of the `ModelManager` class from `django.db.models`.

So to get all of the `User` instances stored in our database we start with the `User` class object. We use the model manager `objects` and call the query api method `all()` on that: `User.objects.all()`.
```
In [3]: User.objects.all()
Out[3]: []
```
We don’t have any users. Let’s make one:
```
In [4]: bob = User(username='bob', email='bob@example.com', password='secret')

In [5]: bob
Out[5]: <User: bob>

In [6]: bob.username
Out[6]: 'bob'

In [7]: bob.password
Out[7]: 'secret'
```
What’s wrong with this? Our password is stored in plaintext. When you set a password directly on Django’s user object, it will store it in plaintext. Instead you should use `.set_password()`:
```
In [8]: bob.set_password('secret')

In [9]: bob.password
Out[9]: u'pbkdf2_sha256$20000$kVSkzjpfrnQa$TUudARpt5n+DRUO8c+rNKNm+Br2VwYoQL5QvgiD6qkA='
```
Now we have a hashed password. This is still a terrible password, but at least it’s not stored in plaintext. This will be important when you run tests, because Django’s authentication systems will be looking for a hashed password.

Bob still doesn’t have an `id`, because we never saved bob:
```
In [10]: bob.id

In [11]: bob.save()

In [12]: bob.id
Out[12]: 1
```

## Relating Users to Profiles

Let’s check our `PatronProfile` objects:
```
In [13]: PatronProfile.objects.all()
Out[13]: <QuerySet [<PatronProfile: bob>]>
```
Bob has an `PatronProfile` already. We talked before about how to hook up a connection that saves an PatronProfile for a user automatically if it doesn’t already exist.

If we want to get the user bob’s profile we can do this:
```
In [14]: PatronProfile.objects.get(user=bob)
Out[14]: <PatronProfile: bob>
```
Notice we’re not passing an id here. What is `bob`? It’s a `User` object. We are asking our `PatronProfile` to give us the profile object who’s user is this `User` object.

Look at our `PatronProfile` definition:

_/lending_library/patron_profile/models.py_
```python
...
@python_2_unicode_compatible
class PatronProfile(models.Model):
    user = models.OneToOneField(
        User,
        related_name="profile",
        null=False
    )
...
```
`PatronProfile` has a `user` attribute that is a `OneToOneField` that references `User` objects. That means we can use `User` objects directly as query values, and we can get back the profile we want.

When working with an ORM, you should try always to think in terms of Python objects. Allow the ORM to handle figuring out how to get from the object to the ID that is actually stored in the database. That is its job.
```
In [15]: bobs_profile = _

In [16]: bobs_profile
Out[16]: <PatronProfile: bob>

In [17]: bobs_profile.user == bob
Out[17]: True

In [18]: bobs_profile.user
Out[18]: <User: bob>

In [19]: bob
Out[19]: <User: bob>
```
These are all equivalent.

## Relationships and Back References

Let’s look at our model again. If you look above to our `PatronProfile` definition, you will see `related_name="profile"`. When we use `related_name`, that means the `User` model at the opposite end of that relationship **will have an attribute added to it that points back to this object**. That means we should be able to ask bob for his ` profile`, and get back an `PatronProfile` object that corresponds to this user.
```
In [20]: bob.profile
Out[20]: <PatronProfile: bob>
```
This is the Django way. All of our relationship fields will have this same kind of built in connection.

Let’s say you don’t want this to exist. Let’s remove it from our `PatronProfile`:

_/lending_library/patron_profile/models.py_
```python
...
@python_2_unicode_compatible
class PatronProfile(models.Model):
    user = models.OneToOneField(
        User,
        null=False
    )
...
```
Now, restart our Django shell:
```
In [1]: from patron_profile.models import PatronProfile

In [2]: profiles = PatronProfile.objects.all()

In [3]: profiles
Out[3]: [<PatronProfile: bob>]

In [4]: bobs_profile = profiles[0]

In [5]: bobs_profile
Out[5]: <PatronProfile: bob>
```
Bob’s profile has a user attribute:
```
In [6]: bob = bobs_profile.user

In [7]: bob
Out[7]: <User: bob>
```
Before we had a `related_name` attribute. Now?
```
In [8]: bob.profile
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
<ipython-input-8-ea648d991c5e> in <module>()
----> 1 bob.profile

AttributeError: 'User' object has no attribute 'profile'
```
We didn’t change anything in the database. That `profile` attribute we saw on `bob` before was created in Python at runtime. All of these reverse relationships are done by Django in the ORM.

Although we’ve not told our application that we want a `related_name`, it still makes one for us by default:
```
In [9]: dir(bob)

...
 'has_perm',
 'has_perms',
 'has_usable_password',
 'id',
 'patronprofile',
 'is_active',
 'is_anonymous',
 'is_authenticated',
 'is_staff',
...
```
Notice the `patronprofile` attribute. Django defaults to taking the name of the model, making it lowercase, and using it as the `related_name`.
```
In [10]: bob.patronprofile
Out[10]: <PatronProfile: bob>
```
Magic! What if we _really_ don’t want that?

_/lending_library/patron_profile/models.py_
```python
...
@python_2_unicode_compatible
class PatronProfile(models.Model):
    user = models.OneToOneField(
        User,
        related_name="+",
        null=False
    )
...
```
You can set the `related_name` attribute to `"+"`. Let’s check if that worked:
```
In [1]: from patron_profile.models import PatronProfile

In [2]: profiles = PatronProfile.objects.all()

In [3]: bobs_profile = profiles[0]

In [4]: bobs_profile
Out[4]: <PatronProfile: bob>

In [5]: bob = bobs_profile.user

In [6]: dir(bob)
Out[6]:
...
 'has_perm',
 'has_perms',
 'has_usable_password',
 'id',
 'is_active',
 'is_anonymous',
 'is_authenticated',
 'is_staff',
...
```
There’s no `PatronProfile` attribute now. Why is the symbol for suppressing the `related_name` `"+"`? That’s what the folks who built Django chose.

The `OneToOneField` field is a useful, but slightly strange relationship. It knows ahead of time that there will only ever be one object on either end. On the other hand, it’s possible for us to create a situation where we don’t have that other object:
```
In [7]: sallys_profile = PatronProfile()

In [8]: sallys_profile.user
---------------------------------------------------------------------------
RelatedObjectDoesNotExist                 Traceback (most recent call last)
<ipython-input-8-cdfd1c2ae232> in <module>()
----> 1 sallys_profile.user

/.../lib/python2.7/site-packages/django/db/models/fields/related.pyc in __get__(self, instance, instance_type)
    614         if rel_obj is None and not self.field.null:
    615             raise self.RelatedObjectDoesNotExist(
--> 616                 "%s has no %s." % (self.field.model.__name__, self.field.name)
    617             )
    618         else:

RelatedObjectDoesNotExist: PatronProfile has no user.
```
We get this `RelatedObjectDoesNotExist` error. You can’t actually import this exception from anywhere. It doesn’t exist in code. It’s a dynamically created subclass of another exception type. Django builds it and uses it to flag a particular kind of failure.

Although you can’t import (and thus can’t catch) that specific exception type, there is a way to deal with it. Remember that if an exception inherits from another exception, an `except` line meant to catch the subclass will also trigger on its parent.

The `User` class (and in fact _any_ Django Model subclass) has an attribute called ` DoesNotExist`:
```
In [10]: from django.contrib.auth.models import User

In [11]: User.DoesNotExist
Out[11]: django.contrib.auth.models.DoesNotExist
```
As it turns out, that exception class is a parent class of this `RelatedObjectDoesNotExist` thing that gets raised. We can catch that exception:
```
In [12]: try:
  ....:     sallys_profile.user
  ....: except User.DoesNotExist:
  ....:     print("phooey, no user")
  ....:
phooey, no user
```
We can catch that exception.

## Relationships and Managers

The `user` attribute we have for our Django model is a `OneToOneField`. I say it’s a bit odd when it comes to relationship fields because it points directly at the other model. When you ask for that attribute, you either get that other instance, or you get this `DoesNotExist` error. For all the other relationships, the field attribute on our model is actually a special type of model manager, a **relationship** manager.

If, for example, we wanted to add a new `friends` attribute to our `PatronProfile` (and return our `related_name` to the previous state):

_/lending_library/patron_profile/models.py_
```python
@python_2_unicode_compatible
class PatronProfile(models.Model):
    user = models.OneToOneField(
        User,
        related_name="profile",
        null=False
    )
...
    friends = models.ManyToManyField(User, related_name="friends")
...
```
We’ve added a new field to our model. We need to `makemigrations` and `migrate` these changes to our DB:
```bash
$ python manage.py makemigrations patron_profile
Migrations for 'patron_profile':
  0003_auto_20160116_0051.py:
    - Add field friends to PatronProfile
    - Alter field user on PatronProfile

$ python manage.py migrate
Operations to perform:
  Synchronize unmigrated apps: staticfiles, gis, messages
  Apply all migrations: lender_books, sessions, admin, patron_profile, auth, contenttypes
Synchronizing apps without migrations:
  Creating tables...
    Running deferred SQL...
  Installing custom SQL...
Running migrations:
  Rendering model states... DONE
  Applying patron_profile.0003_auto_20160116_0051... OK
```
Back in our shell, let’s create a few more users:
```
In [2]: for username in ['sally', 'tom', 'harry', 'jane']:
    user = User(username=username, email='{}@example.com'.format(username))
    user.set_password(username)
    user.save()
   ...:

In [5]: User.objects.all()
Out[5]: <QuerySet [<User: bob>, <User: sally>, <User: tom>, <User: harry>, <User: jane>]>

In [17]: profiles = [p.profile for p in User.objects.all()]

In [18]: profiles
Out[18]:
[<PatronProfile: bob>,
 <PatronProfile: sally>,
 <PatronProfile: tom>,
 <PatronProfile: harry>,
 <PatronProfile: jane>]
```
Now we have a list of profiles that have friends:
```
In [19]: bobs_profile = profiles[0]

In [20]: bobs_profile
Out[20]: <PatronProfile: bob>

In [21]: bobs_profile.friends
Out[21]: <django.db.models.fields.related_descriptors.create_forward_many_to_many_manager.<locals>.ManyRelatedManager at 0x1027a0908>
```
What is the object returned here? It’s a `ManyRelatedManager` instance. Where does this thing come from? We can use the _method resolution order_ of its class to find out:
```
In [21]: bobs_profile.friends.__class__.__mro__
Out[21]:
(django.db.models.fields.related_descriptors.create_forward_many_to_many_manager.<locals>.ManyRelatedManager,
 django.contrib.auth.models.UserManager,
 django.contrib.auth.base_user.BaseUserManager,
 django.db.models.manager.Manager,
 django.db.models.manager.BaseManagerFromQuerySet,
 django.db.models.manager.BaseManager,
 object)
```
Interesting. It actually inherits from the `UserManager`. We’ve seen one of those before, haven’t we?
```
In [22]: User.objects
Out[22]: <django.contrib.auth.models.UserManager at 0x105f63650>
```
This is a `UserManager`. So our manager object has the same methods that our `UserManager` has. For example, `.all()`
```
In [23]: bobs_profile.friends.all()
Out[23]: <QuerySet []>
```
Let’s take a slice of our users:
```
In [24]: users = User.objects.all()

In [25]: users
Out[25]: <QuerySet [<User: bob>, <User: sally>, <User: tom>, <User: harry>, <User: jane>]>

In [27]: users[2:5]
Out[27]: <QuerySet [<User: tom>, <User: harry>, <User: jane>]>
```
And add that slice as friends of bob:
```
In [28]: bobs_profile.friends.add(*users[2:5])

In [29]: bobs_profile.friends.all()
Out[29]: [<User: tom>, <User: harry>, <User: jane>]
```
Notice one thing. Let’s restart our Django shell:
```
In [1]: from django.contrib.auth.models import User

In [2]: users = User.objects.all()

In [3]: users
Out[3]: [<User: bob>, <User: sally>, <User: tom>, <User: harry>, <User: jane>]

In [4]: bob = users[0]

In [5]: bobs_profile = bob.profile

In [6]: bobs_profile
Out[6]: <PatronProfile: bob>

In [7]: bobs_profile.friends.all()
```
What will this `bobs_profile` friends list contain now?
```
Out[7]: [<User: tom>, <User: harry>, <User: jane>]
```
There’s an implication here. Saving happens automatically. What this means is that the `save()` method is not called. Any customizations you might make to the `save()` method of these models will be skipped entirely. Signal handlers that are wired to listen for `save()` signals don’t happen. You need to be aware of this.

## Summary

We have relationship fields. The majority of relationship fields work in a way that the attribute on the object that represents that relationship field is actually a model manager. That manager will hand you back things related to the object you are using. It’s like a pre-filtered selection of objects from the other end of the relationship. It will only ever speak to objects that happen to be related to the object you are looking at now.

The one exception is our `OneToOneField`. It operates by handing back the one object, or raising a particular kind of error (`DoesNotExist`).

