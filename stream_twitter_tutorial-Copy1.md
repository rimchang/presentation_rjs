Build a scalable Twitter clone with Django and GetStream.io
---------------------------------------------------

In this tutorial we are going to build a Twitter clone using Django and [GetStream.io](https://GetStream.io), a hosted API for newsfeed development.
We will show you how easy is to power your newsfeeds with GetStream.io. At the end of this tutorial we will have a Django app with a profile feed, a timeline feed, support for following users, hashtags and mentions.

I assume that you are familiar with Django. If you're new to Django the [official tutorial] (https://docs.Djangoproject.com/en/1.7/intro/install/) explains it very well.


Bootstrap the Django application
--------------------------------

We will use Python 2.7 and Django 1.7, which is the latest major release at the time of writing.
Make sure you have a working Django project before you continue to the next part of the tutorial.


Create the Django app
---------------------
Let's start by creating a new Django app called stream_twitter

```sh
python manage.py startapp stream_twitter
```


Install stream_django
---------------------

[stream_django](http://github.com/getstream/stream-django) provides the GetStream integration for Django, it is built on top of the [stream_python](https://github.com/getstream/stream-python) client.

```sh
pip install stream_django
```

To enable stream_django you need to add it to your INSTALLED_APPS:

```python
INSTALLED_APPS = (
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'stream_twitter',
    'stream_django'
)
```

GetStream.io setup
------------------

First of all, we need to create an account on GetStream.io. You can signup with Github and it's free for usage below 3 million feed updates per month. Once you've signed up, get your api_key and api_secret from the dashboard and add them to Django's settings:

```python
STREAM_API_KEY = 'my_api_key'
STREAM_API_SECRET = 'my_api_secret'
```


The models
----------

In this application we will have three different models: users, tweets and follows.

To keep it as simple as possible, we will use Django's contrib.auth User model.
Have a look below at the initial version of the Tweet and Follow models.

```python
class Tweet(models.Model):
    user = models.ForeignKey('auth.User')
    text = models.CharField(max_length=160)
    created_at = models.DateTimeField(auto_now_add=True)


class Follow(models.Model):
    user = models.ForeignKey('auth.User', related_name='friends')
    target = models.ForeignKey('auth.User', related_name='followers')
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        unique_together = ('user', 'target')
```

Now, let's create the schema migrations and apply them.

```sh
python manage.py makemigrations stream_twitter
python manage.py migrate
```

Let's also setup the view to add tweets.

```python
from django.views.generic.edit import CreateView
from stream_twitter.models import Follow
from stream_twitter.models import Tweet


class TweetView(CreateView):
    model = Tweet
    fields = ['text']

    def form_valid(self, form):
        form.instance.user = self.request.user
        return super(Tweet, self).form_valid(form)
```

And of course add it to urls.py

```
from django.conf.urls import patterns, include, url
from django.contrib import admin
from django.contrib.auth.decorators import login_required
from stream_twitter import views


urlpatterns = patterns('',
    url(r'^admin/', include(admin.site.urls)),
    url(r'^tweet/', login_required(views.TweetView.as_view()))
)
```

Now that we have the view setup for creating tweets we can move on to setting up the newsfeed.

Model integration
-----------------

We want the tweets to be stored in the authors feed. This is when we start using the stream_django integration. We can configure the Tweet model so that it will syncronise automatically with feeds.

To do this we need to make Tweet a subclass of ```stream_django.activity.Activity```

```python
from stream_django import activity


class Tweet(activity.Activity, models.Model):
    ...
```

From now on, new tweets will be added to the user feed of the author *and* to the feeds of all his followers. The same applies to deleting a tweet.

So, let's give it a try using Django's shell:

```sh
python manage.py shell
```

```python
from stream_twitter.models import Tweet
from django.contrib.auth.models import User

user = User.objects.all().first()

Tweet.objects.create(
    user=user,
    text='Go Cows!'
)
```

```
NotImplementedError: Tweet must implement activity_object_attr property
```

Oh snap! Why is Django complaining about some missing property in the Tweet model?

The reason for this is quite simple and the problem is easy to fix. When the Tweet is created, stream_django converts it into an activity and sends that to GetStream APIs.

Now, this is the first time we talk about activities, let's take a moment to define what an activity is. 

An activity is an object that contains the information about an action that is performed by someone involving an object. When you write data to GetStream's feeds, you send this data in the form of activities. The simplest activity is made by these three fields: actor, object and verb. For example: Tommaso tweets: 'Go cows!'

GetStream.io API's allow you to store additional fields in your feeds, as you can see from the documentation [here](https://GetStream.io/docs/#add-remove-activities).

In order to convert a model instance into an activity, the model class must implement `activity_object_attr`. In the case of Tweet, the *object* is the tweet itself, the *actor* is the user that created the tweet and the *verb* is 'tweet'.

```python
class Tweet(activity.Activity, models.Model):
    ...

    @property
    def activity_object_attr(self):
        return self
```

The same code that we used before will work now, meaning that the tweet gets stored on the author's feed. We can verify that by using the Data Browser in GetStream.io's Dashboard.

The tweet data will be stored in the feed with *slug* user and *id* 1

![Dashboard](http://oi60.tinypic.com/2rhmvtk.jpg)

User feed
---------

So now that every tweet gets stored in the author's feed, we are going to add a view that reads them.

```python
from django.contrib.auth.models import User
from stream_django.enrich import Enrich
from stream_django.feed_manager import feed_manager


def profile_feed(request, username=None):
    enricher = Enrich()
    user = User.objects.get(username=username)
    feed = feed_manager.get_user_feed(user.id)
    activities = feed.get(limit=25)['results']
    enricher.enrich_activities(activities)
    context = {
        'activities': activities
    }
    return render(request, 'tweets.html', context)
```


There are two new things that I should explain: the *feed manager* and the *enricher*. As the name suggests, the feed manager takes care of managing the different feeds involved in your app. In this case we ask the feed manager to give us the user feed for current user.

We learned before that data is stored in feeds in form of activities. This is what a tweet looks like when we read it from GetStream.io:

```
[{
    'actor': 'auth.User:1',
    'object': 'stream_twitter.Tweet:1',
    'verb': 'tweet',
    ... other fields ...
}]
```

As you can see, 'object' field does not contain the tweet itself but a reference to that (the same applies to the 'actor' field). The enricher replaces these references with model instances.

Templating feeds
----------------

Django_stream comes with a templatetag that helps you to show the content from feeds in your templates. This can get quite complex as you add different kinds of activities in your feeds.

Here is a very minimal tweets.html template:

```htmlDjango
{% load activity_tags %}

{% for activity in activities %}
    {% render_activity activity %}
{% endfor %}
```

The first time your run this, Django will complain that 'activity/tweet.html' is missing. That's because the render_activity templatetag inspects the activity object and loads the template based on the verb.
Because the verb in this case is 'tweet', it will look for tweet.html in activity path. The templatetag accepts extra options to make your templates as re-usable as possible, see [here](https://github.com/GetStream/stream-Django#templating) for the templatetag documentation.


Feed Follow
-----------

As a next step we're going to add the ability to follow users to the application. To do this we create a view that creates Follow objects.

```python
class FollowView(CreateView):
    model = Follow
    fields = ['target']

    def form_valid(self, form):
        form.instance.user = self.request.user
        return super(Tweet, self).form_valid(form)
```

Now we can use Django's signals to perform follow/unfollow requests on GetStream APIs.

```python
def unfollow_feed(sender, instance, **kwargs):
    feed_manager.unfollow_user(instance.user_id, instance.target_id)


def follow_feed(sender, instance, created, **kwargs):
    if created:
        feed_manager.follow_user(instance.user_id, instance.target_id)


post_save.connect(follow_feed, sender=Follow)
post_delete.connect(unfollow_feed, sender=Follow)
```

Timeline view (AKA flat feed)
-----------------------------

The hardest part for a scalable Twitter clone is the feed showing the tweets from people you follow. This is commonly called the timeline view or newsfeed. The code below shows the timeline.

```python
def timeline(request):
    enricher = Enrich()
    feed = feed_manager.get_news_feeds(request.user.id)['flat']
    activities = feed.get(limit=25)['results']
    enricher.enrich_activities(activities)
    context = {
        'activities': activities
    }
    return render(request, 'timeline.html', context)
```

The code looks very similar to the code of profile_feed. The main difference is that we use feed manager's get_news_feeds. By default, GetStream.io apps and stream_django come with two newsfeeds predefined: flat and aggregated feeds. When you use `feed_manager.get_news_feeds`, you get a dictionary with flat and aggregated feeds. Since we are not going to use aggregated feeds, we can adjust Django settings:

```python
STREAM_NEWS_FEEDS = dict(flat='flat')
```

Hashtags feeds
--------------

We want Twitter style hashtags to work as well. Doing this is surprisingly easy. Let's first open GetStream.io dashboard and create the 'hashtag' feed type. (Note: By default getstream.io will setup user, flat, aggregated and notification feeds. If you more feeds you need ot configure them in the dashboard)

![hashtagfeedform](http://oi57.tinypic.com/68akah.jpg)

```python
class Tweet(activity.Activity, models.Model):

    def parse_hashtags(self):
        return [slugify(i) for i in self.text.split() if i.startswith("#")]
```

Now that we have parsed the hashtags, we could loop over them and publish the same activity to every hashtag feed. Fortunately there's a shortcut though. GetStream allows you to send a copy of an activity to many feeds with a single request.

To do this, we only need to implement the `activity_notify` method in Twitter model:

```python
class Tweet(activity.Activity, models.Model):

    @property
    def activity_notify(self):
        targets = []
        for hashtag in self.parse_hashtags():
            targets.append(feed_manager.get_feed('hashtag', hashtag))
        return targets
```

From now on, activities will be stored to hashtags feeds as well. For instance, the feed 'hashtag:Django' will contain all tweets with '#Django' in it.

Again the view code looks really similar to the other views.

```python
def hashtag(request, hashtag):
    enricher = Enrich()
    feed = feed_manager.get_feed('hashtag', hashtag)
    activities = feed.get(limit=25)['results']
    enricher.enrich_activities(activities)
    context = {
        'activities': activities
    }
    return render(request, 'hashtag.html', context)
```

Mentions
--------

Now that we found out about the `activity_notify` property, it only takes a bunch of extra lines of code to add user mentions.

```python

class Tweet(activity.Activity, models.Model):

    def parse_mentions(self):
        mentions = [slugify(i) for i in self.text.split() if i.startswith("@")]
        return User.objects.filter(username__in=mentions)

    @property
    def activity_notify(self):
        targets = []
        for hashtag in self.parse_hashtags():
            targets.append(feed_manager.get_feed('hashtag', hashtag))
        for user in self.parse_mentions():
            targets.append(feed_manager.get_news_feeds(user.id)['flat'])
        return targets
```

Conclusion
----------
Congratulations, you've reached the end of this tutorial. This article showed you how easy it is to build scalable newsfeeds with Django and GetStream.io. It took us just 100 LoC and (I hope) less than one hour to get this far.

You can find the code from this tutorial and the fully functional application on [GitHub](https://github.com/GetStream/django_twitter). The same application is also available online [here](http://tw.getstream.io/). I hope you found this interesting and useful and I'd be glad to answer all of your questions.

If you're new to Django or GetStream.io, I highly recommend the [official django tutorial](https://docs.Djangoproject.com/en/1.7/intro/install/) and the [getstream.io getting started](https://getstream.io/get_started/#intro).