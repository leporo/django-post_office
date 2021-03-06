==================
Django Post Office
==================

Django Post Office is a simple app that allows you to send email asynchronously
in Django. Supports HTML email, database backed templates and logging.

``post_office`` is implemented as a Django ``EmailBackend`` so you don't need to
change any of your code to start sending email asynchronously.


Dependencies
============

* `django >= 1.2 <http://djangoproject.com/>`_


Installation
============

.. image:: https://travis-ci.org/ui/django-post_office.png?branch=master


* Install from PyPI (or you can `manually download it from PyPI <http://pypi.python.org/pypi/django-post_office>`_)::

    pip install django-post_office

* Add ``post_office`` to your INSTALLED_APPS in django's ``settings.py``:

.. code-block:: python

    INSTALLED_APPS = (
        # other apps
        "post_office",
    )

* Run ``syncdb``::

    python manage.py syncdb

* Set ``post_office.EmailBackend`` as your ``EMAIL_BACKEND`` in django's ``settings.py``::

    EMAIL_BACKEND = 'post_office.EmailBackend'


Usage
=====

If you already use Django's ``send_mail`` command and want to send out emails
asynchronously, simply set ``post_office`` as your email backend in
``settings.py`` and it will automatically queue outgoing emails without you
needing to change anything::

    EMAIL_BACKEND = 'post_office.EmailBackend'

To actually send them out, run ``python manage.py send_queued_mail``. You can schedule this
to run regularly via cron::

    * * * * * (/usr/bin/python manage.py send_queued_mail >> send_mail.log 2>&1)


By default, ``post_office`` uses django's SMTP ``EmailBackend``. If you want to
use a different backend, you can do so by changing ``POST_OFFICE_BACKEND``.

For example if you want to use `django-ses <https://github.com/hmarr/django-ses>`_::

    POST_OFFICE_BACKEND = 'django_ses.SESBackend'

You can view also queued emails along with their statuses if you have django's
admin interface enabled:

.. code-block:: python

    INSTALLED_APPS = (
        # ...
        'django.contrib.admin',
        # ...
    )

Sending HTML Emails
-------------------

``post_office`` also comes with a ``send_mail`` command similar to django's.
It accepts two extra arguments, ``html_message`` and
``priority`` (``high``, ``medium``, ``low`` or ``now``).

Here's how to use it:

.. code-block:: python

    from post_office import send_mail, PRIORITY
    send_mail('subject', 'plaintext message', 'from@example.com', ['to@example.com'],
              '<p>HTML message</p>', priority=PRIORITY.medium)

``post_office`` is also task queue friendly. Passing ``now`` as priority into
``send_mail`` will deliver the email right away, regardless of how many emails
you have in your queue:

.. code-block:: python

    from post_office import send_mail, PRIORITY
    send_mail('subject', 'plaintext message', 'from@example.com', ['to@example.com'],
              '<p>HTML message</p>', priority=PRIORITY.now)

This is useful if you already use something like `django-rq <https://github.com/ui/django-rq>`_
to send emails asynchronously and only need to store email activities and logs.


Email Templates
---------------

``post_office`` also allows you to easily send template rendered emails.
Email templates in ``post_office`` are stored in database and can be easily
added via Django's ``admin`` interface.

Here's how to send templated emails:

1. Create an ``EmailTemplate`` from django's ``admin`` interface
2. Send the email using ``send_templated_mail`` command:

.. code-block:: python

    from post_office.utils import send_templated_mail
    send_templated_mail('template_name', 'from@example.com', ['to@example.com'])

    # Here's a list of full arguments accepted by send_templated mail
    send_templated_mail(
        'template_name',        # Name of the template
        'from@example.com',     # Sender email
        ['to@example.com'],     # List of recipient emails
        context={'foo': 'bar'}, # Extra data that will be used during template rendering
        priority=PRIORITY.now,  # Email priority
    )

Template Tags and Variables
---------------------------

You can, of course use Django's template tags and variables when in templates.
For example, if you put "Hello, {{ name }}" in the subject line and pass in 
``{'name': 'Alice'}`` as context, you will get "Hello, Alice" as subject:

.. code-block:: python
    
    from post_office.models import EmailTemplate
    from post_office.utils import send_templated_mail

    EmailTemplate.objects.create(
        name='morning_greeting',
        subject='Morning, {{ name|capfirst }}',
        content='Hi {{ name }}, how are you feeling today?',
        html_content='Hi <b>{{ name }}</b>, how are you feeling today?',
    )

    send_templated_mail(
        'morning_greeting',
        'from@example.com',
        ['to@example.com'],
        context={'name': 'alice'},
    )

    # This will create an email with the following content:
    subject = 'Morning, Alice',
    content = 'Hi alice, how are you feeling today?'
    content = 'Hi <strong>alice</strong>, how are you feeling today?'

``post_office`` will cache ``EmailTemplate``s by default if Django's caching
mechanism is configured. If for some reason you want to disable caching, you can
set ``POST_OFFICE_CACHE`` to ``False`` in ``settings.py``:

.. code-block:: python

    ## All cache key will be prefixed by post_office:template:
    ## To turn OFF caching, you need to explicitly set POST_OFFICE_CACHE to False in settings
    POST_OFFICE_CACHE = False

    ## Optional: to use a non default cache backend, add a "post_office" entry in CACHES
    CACHES = {
        'post_office': {
            'BACKEND': 'django.core.cache.backends.memcached.PyLibMCCache',
            'LOCATION': '127.0.0.1:11211',
        }
    }


Management Commands
-------------------

* ``send_queued_mail`` - send queued emails, those that aren't successfully
  sent they will be marked as ``failed``.

* ``cleanup_mail`` - delete all emails created before an X number of days
  (defaults to 90).

You may want to set these up via cron to run regularly::

    * * * * * (cd $PROJECT; python manage.py send_queued_mail >> $PROJECT/cron_mail.log 2>&1)
    0 1 * * * (cd $PROJECT; python manage.py cleanup_mail --days=30 >> $PROJECT/cron_mail_cleanup.log 2>&1)


Testing
=======


To run ``post_office``'s test suite::

    `which django-admin.py` test post_office --settings=post_office.test_settings --pythonpath=.


Changelog
=========

Version 0.2.2
-------------
* ``created`` fields now use ``auto_now_add``
* ``last_updated`` fields now use ``auto_now``

Version 0.2.1
-------------
* Fixed typo in ``admin.py``

Version 0.2
-----------
* Allows sending emails via database backed templates

Version 0.1.5
-------------
* Errors when opening connection in ``Email.dispatch`` method are now logged
