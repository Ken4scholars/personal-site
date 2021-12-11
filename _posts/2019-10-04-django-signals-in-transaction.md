---
layout: post
section-type: post
title: Django Post-save Signal in a Transaction
category: Django-Tricks
tags: [ 'Django', 'Django-Signals', 'atomic-transactions', 'pubsub']
---

So let's assume you have this cool Django project that uses <a href="https://docs.djangoproject.com/en/2.2/topics/signals/" target="\_blank">signals</a>
to achieve loose coupling between different parts of your code and across apps. There's been
quite a lot of arguments against signals but you will agree with me that it does make things
clean especially when it comes to communication across apps. The major argument against it being
that it makes code less readable as it is not readily obvious where some action is performed 
after a signal is fired. This is quite true, especially when you start working on an existing project 
and you have to navigate through the code to find out where a particular functionality is implemented.
Direct function calls are usually more obvious and readable but can also lead to too much dependency and coupling
between different parts of your code. The Django signals system is an implementation of the popular
<a href="https://abdulapopoola.com/2013/03/12/design-patterns-pub-sub-explained/" target="\_blank">publisher/subscriber</a> (or just pubsub) design pattern
so it is actually nothing new nor is it limited to the framework. The issue is that because most of popular signals (say hello to <a href="https://docs.djangoproject.com/en/2.2/ref/signals/#django.db.models.signals.post_save" target="\_blank">post_save</a>)
are inbuilt and triggered by the framework, it is usually not obvious when they are used because it is not visible in your code.

Now, let's also assume that you are using <a href="https://docs.djangoproject.com/en/2.2/topics/db/transactions/#controlling-transactions-explicitly" target="\_blank">atomic transactions</a>
somewhere in your views or business logic. Now you don't just have a cool code, you have a hipster code!

<img alt="Hipster Coder!" src="{{site.baseurl}}/img/hipster_coder.jpg">

What transaction does is guarantee <a href="https://en.wikipedia.org/wiki/ACID" target="\_blank">ACIDity</a> (yes, I know you definitely frowned at that!) 
of DB queries in a block of code and ensures that a rollback can be done for the whole block in case of an exception. 

*Lots of talk! Throw us some code now, will ya?*

<pre><code data-trim class="python">
{% raw %}
def register(username, password):
    with transaction.atomic():
       user = User(username=username)
       user.set_password(password)
       
       user.save()
       # post_save is triggered here
       
       # do some other stuff that could 
       # raise exceptions or just break stuff

       code_that_can_raise_exception(user)
       
       return user
{% endraw %}    
</code></pre>

Now great, there you have it. So we have wrapped our user registration block (you could also decorate the entire method)
with transaction.atomic to ensure that if anything happened during the registration, we could easily rollback and 
remove the user's footprint from the db and not leave any stale records.  

Now we have a `post_save` handler for User that say sends a welcome notification

<pre><code data-trim class="python">
{% raw %}
@receiver(post_save, sender=User, dispatch_uid='send_welcome_notification')
def send_welcome_email(sender, instance, created, **kwargs):
    if created:
        send_email(
            subject='Welcome Email',
            message='Welcome to our hipster service! You can now login!',
            from_email='noreply@hipster-service.com',
            recipient_list=[instance.email]
        )
{% endraw %}    
</code></pre>

Now we can all shout hurray! and go for some coffee right? Well sorry to disappoint you honey, but just not yet.
This all works fine if the transaction block above always exits successfully without `code_that_can_raise_exception` actually
raising an exception. The problem is when it eventually does. The DB operations are rolled back and the 
user is not committed in the DB so he basically doesn't exist. But wait...our `post_save` should have been fired as soon
as `user.save()` was called right? which means our signal handler eventually sent the welcome email
even though the user wasn't registered and that means he can't login as we promised. Damn, that's bad. Transactions are bad and I hate them. Are they?

Actually, if all the signal handler did was change some other object in the DB, things would still be good as it would be 
successfully rolled back too along with any other branches of the code that are within the transaction. But when you are running a task like
sending emails or sending request to an external service, this can't be reversed or rolled back, so this is where the main issue is.

In my workplace the case was indexing the newly created object in Elasticsearch using Haystack's 
<a href="https://django-haystack.readthedocs.io/en/master/signal_processors.html#realtime-realtimesignalprocessor" target="\_blank">RealtimeSignalProcessor</a>.
The objects were not committed due to the rollback but were indexed in ES and our search functionality powered by ES sometimes returned 
these orphaned objects that don't exist in the DB.

To solve this problem, we used Django's <a href="https://docs.djangoproject.com/en/1.11/topics/db/transactions/#performing-actions-after-commit" target="\_blank">transaction.on_commit</a> hook in a custom Haystack signal processor.
The hook basically allows you to pass in callbacks that can be executed after the current transaction is committed. If there is no transaction currently open, then 
the callback is executed immediately. The callback should be a simple callable that takes no arguments so if arguments are needed, the function can be wrapped in a lambda function

For the welcome email example above, we would have something like this.


<pre><code data-trim class="python">
{% raw %}
@receiver(post_save, sender=User, dispatch_uid='send_welcome_notification')
def send_welcome_email(sender, instance, created, **kwargs):
    if created:
        transaction.on_commit(lambda: send_email(
            subject='Welcome Email',
            message='Welcome to our hipster service! You can login!',
            from_email='noreply@hipster-service.com',
            recipient_list=[instance.email]
        ))
{% endraw %}    
</code></pre>

This means that if the `code_that_can_raise_exception` eventually raises an exception, then the transaction is rolled back and subsequently,
the `send_email` on commit callback would not be called as the transaction was never committed. Great right? Now you can go have your coffee:)

In conclusion, be careful when having code that could fire a signal in a transaction block as the transaction could be rolled
back but the signal could already be fired before that. In many cases, the signal handler would just be making some other changes to the
DB which would equally be rolled back, so there is no issue there. The real issue is when you are making external calls like writing something to a file, calling an external service etc.

Chau!