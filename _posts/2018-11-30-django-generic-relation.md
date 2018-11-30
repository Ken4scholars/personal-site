---
layout: post
section-type: post
title: Django Generic Relation - multiple usage in the same model
category: Django Tricks
tags: ['Python', 'Django', 'Database models', 'Generic Relations']
---

If you have developed web services using <a href="https://djangoproject.com" target="\_blank">Django,</a> chances are that you 
have come across the <a href="https://docs.djangoproject.com/en/2.1/ref/contrib/contenttypes/#generic-relations" target="\_blank">Generic Relations.</a>
It is a technique that allows you to link a model with multiple other models without knowing them before hand. It comes 
in quite handy when you have common traits which can be applied to several models. A typical example is a _tagging_ system
which is used as an example in the docs. 

I have used it a couple times and found it very useful, however, there is one problem I have noticed with it - you cannot 
apply that trait (say tags) more than once in a model. To illustrate this issue and how I solved it, let's look at my 
example.

First I had a model `Event` which has images which I process using <a href="https://github.com/bradleyg/django-s3direct"> django-s3direct.</a>

<pre><code data-trim class="python">
{% raw %}
class Event(models.Model):
    ...
    images = JSONField(S3DirectField(), null=True, blank=True)
    ...
{% endraw %}
</code></pre>

The first problem was that using the JSONField, a lot of work had to be done in the admin to be able to display 
the s3direct file upload widget correctly. As a result, I decided to create a separate model to wrap the s3direct field 
and use the widget provided by the package. Also, considering that I will be using images in other models too, I decided 
to use generic relations. 

So now we have this:

<pre><code data-trim class="python">
{% raw %}
class Image(models.Model):
    link = S3DirectField()

    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    content_object = GenericForeignKey('content_type', 'object_id')

class Event(models.Model):
    ...
    images = GenericRelation(Image, related_name='events')
    ...
    
 class Place(models.Model):
    ...
    images =  GenericRelation(Image, related_name='places')
    ...
{% endraw %}    
</code></pre>

Great! Now the Image model can be used in any model needing images and I can simply do this `event.images.all()` to fetch all the images belonging
to a particular event.
Generally, `GenericRelation` is just a manager so we can use the `images` field the same way we can use any manager: 
`filter()`, `count()`, `first()` etc.

Now the next problem - I suddenly now needed to have images uploaded by the event creators (official images) and images uploaded 
by people who attended the event (let's just say random images). So how do we separate this? surely I can't add another 
`GenericRelation` in the `Event` model because it doesn't even make sense. In fact, `images` is not 
actually a database field but just a manager and having the same manager twice won't make any difference. 
  
So how then can I use this so that I can differentiate between the _official images_ and  _random images_ for events?
The solution is quite simple but may not be very obvious at the beginning (at least it wasn't to me :smile:)

The hack is to do this in the level of the `Image` model. Precisely, I introduced a nullable field `purpose` in the image model that
allows models consuming the relationship to specify the usage of particular `Image` instances.

So now the code looks like this:

<pre><code data-trim class="python">
{% raw %}
class Image(models.Model):
    link = S3DirectField()
    
    # A hack to allow models have "multiple" image fields
    purpose = models.CharField(null=True, blank=True)
    
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveIntegerField()
    content_object = GenericForeignKey('content_type', 'object_id')

class Event(models.Model):
    ...
    images = GenericRelation(Image, related_name='events')
    ...
    
    @property
    def official_images(self):
        return self.images.filter(purpose='official')
    
    @property
    def random_images(self):
        return self.images.filter(purpose='random')
{% endraw %}    
</code></pre>

So to create an official image, all I need do is `event.images.create(link='my link', content_object=event, purpose='official')`.
And I can get them by `event.official_images.all()` <br/>
Of course, the `purpose` field is nullable so other models like `Place` can keep using it the same way if multiple image 
fields are not needed.

There may also be other ways to solve this issue and you can propose yours in the comments. In any case, I do hope this was of help to you. 