Event Tracking
==============

Prerequisites
-------------

It's useful to read both the :ref:`quick_start` and
:ref:`basics` sections for a primer on terminology and concepts.

Using ``pghistory.track``
-------------------------

`pghistory.track` is the primary way to configure
event tracking on models. For example, let's say that we have
the following model:

.. code-block:: python

    class TestModel(models.Model):
        int_field = models.IntegerField()
        char_field = models.CharField(max_length=16, db_index=True)
        user = models.ForeignKey("auth.User", on_delete=models.CASCADE)

Now let's track snapshots for every insert and update of ``TrackedModel``
with `pghistory.track`:

.. code-block:: python

    import pghistory

    @pghistory.track(
        pghistory.Snapshot("snapshot")
    )
    class TrackedModel(models.Model):
        ...

Here's what's happening under the hood:

1. The `pghistory.Snapshot` tracker registers triggers on ``TrackedModel``
   that perform the change tracking. Triggers are installed and configured
   using `django-pgtrigger <https://github.com/Opus10/django-pgtrigger>`__.
2. Changes are written to an auto-generated model. By default, the model will
   be named ``TrackedModelEvent``. It contains every field in ``TrackedModel``
   plus a few additional tracking fields.

Running ``python manage.py makemigrations`` will produce migrations for both
the triggers and the event model.

This is what the generated event model looks like by default:

.. code-block:: python

    class TrackedModelEvent(pghistory.models.Event):
        # The primary key
        pgh_id = models.AutoField(primary_key=True)

        # A foreign key to the tracked model
        pgh_obj = models.ForeignKey(TrackedModel, on_delete=models.DO_NOTHING, related_name='event', db_constraint=False)

        # An identifying label for the event. The first argument to the tracker
        pgh_label = models.TextField()

        # Additional context about the event stored in JSON in the pghistory.Context model
        pgh_context = models.ForeignKey('pghistory.Context', null=True, on_delete=models.DO_NOTHING, related_name='+', db_constraint=False)

        # When the event was created
        pgh_created_at = models.DatetimeField(auto_now_add=True)

        # These fields are copied from the original model. Primary keys and non-foreign key
        # indices are removed. Foreign keys are unconstrained.
        id = models.IntegerField()
        int_field = models.IntegerField()
        char_field = models.CharField(max_length=16)
        user = models.ForeignKey("auth.User", on_delete=models.DO_NOTHING, db_constraint=False)

Here are some important notes about the event models:

* Event models are dynamically created inside the same module as the tracked model
  when using `pghistory.track`. Although event models aren't in models.py,
  they're still imported, migrated, and accessed like a normal model.
* All ``pgh_*`` fields store additional tracked metadata. We'll discuss the ``pgh_context``
  field in greater detail in the `context tracking <TODO>`__ section.
* All other fields are copies of the fields from the tracked model with a few modifications.
  For example, foreign keys are unconstrained and non-foreign key indices are not included.
* The model name and default field/foreign key behavior can be configured
  globally or for each event event model. We'll cover a few options here and discuss it in depth in the
  `configuration <TODO>`__ section.

`pghistory.track` can take multiple trackers that use the same event model. It also takes several
configuration parameters for the generated event model, such as:

* ``fields``: Provide a list of fields to track, otherwise all fields are tracked. If fields are
  provided, the default model name changes, only those fields are tracked, and snapshots are only
  created when those fields change.
* ``exclude``: Track every field but these. ``fields`` and ``exclude`` are mutually exclusive.
* ``model_name``: The name of the generated model. If all fields are tracked, defaults to
  ``<OriginalModelName>Event``.

There are other parameters for configuring ``pgh_*`` fields (``obj_field``, ``context_field``, and ``context_id_field``)
that we will discuss in the `configuration <TODO>`__ section. There's also an ``app_label`` field
for `configuring trackers on third party models <TODO>`__. The final parameters are for low-level
configuration of the event model (``meta``, ``base_model``, and ``attrs``).
See `pghistory.track` for all arguments.

.. note::

    One can also explicitly define the event model without using `pghistory.track`. This is covered
    in the `Custom event models <TODO>`__ section.

We'll wrap up this section with an example of our snapshots in action. Below we create a ``TrackedModel``,
update it, and print the resulting event values:

.. code-block:: python

    from myapp.models import TrackedModel

    m = TrackedModel.objects.create(int_field=1, text_field="hello")
    m.int_field = 2
    m.save()

    # "event" is the default related name of the event model
    print(m.event.values("pgh_obj", "int_field"))

    > [{'pgh_obj': 1, 'int_field': 1}, {'pgh_obj': 1, 'int_field': 2}]

Conditional Tracking
--------------------

In some cases, one may wish to track changes when specific field transitions happen, for example,
storing email addresses every time a user's email changes. Similarly it may not be desirable
to track changes for every model and instead only track changes to "active" ones.

``django-pghistory`` comes with several other trackers for these use cases, all of which accept
a ``condition`` as an argument. Let's take our example of storing user email changes:

.. code-block::

    import pghistory
    import pgtrigger

    @pghistory.track(
        pghistory.BeforeUpdate(
            "email_changed",
            condition=pgtrigger.Q(old__email__df=pgtrigger.F("new__email"))
        ),
        fields=["email"],
        model_name="UserEmailHistory"
    )
    class MyUser(models.Model):
        username = models.CharField(max_length=128)
        email = models.EmailField()

There are two key things going on here:

1. The `pghistory.BeforeUpdate` tracker runs before updates of ``MyUser``, storing
   what the row looked like right before any update happens.
2. We set a condition to only run this tracker when the old email is distinct from
   the new email. I.e. the email has been changed in the update.

``django-pghistory`` uses `django-pgtrigger <https://github.com/Opus10/django-pgtrigger>`__
to register triggers. We've used the ``pgtrigger.Q`` and ``pgtrigger.F`` objects
to create a condition on the old and new email values of the rows.
See the `django-pgtrigger docs <https://django-pgtrigger.readthedocs.io>`__ to learn
more about trigger conditions.

We've named our event model ``UserEmailHistory``, and it only stores the ``email`` field
of the ``MyUser`` model. Let's see what this looks like:

.. code-block::

    from myapp.models import MyUser, UserEmailHistory

    u = MyUser.objects.create(username="hello", email="hello@hello.com")

    # Events are only tracked on updates, so nothing has been stored yet
    assert not UserEmailHistory.objects.exists()

    # Change the email. An event should be stored
    u.email = "world@world.com"
    u.save()
    print(UserEmailHistory.objects.filter(pgh_obj=u).values_list("email", flat=True))

    > ["hello@hello.com"]

Above we create a user and change the email. ``UserEmailHistory`` retains a running
log of all of the previous emails for a user.

There are several core trackers that work like this, all of which run during
different database operations: `pghistory.AfterInsert`, `pghistory.AfterInsertOrUpdate`,
`pghistory.BeforeUpdate`, `pghistory.AfterUpdate`, and `pghistory.BeforeDelete`.

If you need to configure more attributes of the underlying trigger outside of just the condition,
inherit the `pghistory.DatabaseEvent` tracker and use the ``when``, ``operation``, and ``condition``
arguments. These arguments directly correspond to the trigger attributes allowed
by `django-pgtrigger <https://github.com/Opus10/django-pgtrigger>`__. Set the ``snapshot`` attribute
to either ``OLD`` or ``NEW`` to store the old or new row.

.. note::

    `pghistory.Snapshot` already sets a condition to run only when the tracked fields have changed
    and therefore doesn't allow the ``condition`` argument.

Manual Tracking
---------------

Sometimes it is not possible to express an event based on a series
of changes to a model. Some use cases, such as backfilling data, also
require that events are manually created. 

`pghistory.create_event` can be used to manually create events.
Events can be created for existing trackers, or the `pghistory.Event`
tracker can register events that can only be manually created.

Here we register a `pghistory.Event` tracker and manually create
an event:

.. code-block:: python

    @pghistory.track(
        pghistory.Event('user.create'),
        fields=['username']
    )
    class MyUser(models.Model):
        username = models.CharField(max_length=64)
        password = models.PasswordField()

    # Create a user and manually create an "user.create" event
    user = MyUser.objects.create(...)
    pghistory.create_event(user, label='user.create')

.. note::

    Manually-created events will still be linked with context if
    context tracking has started. More on context tracking
    in the `Context section <TODO>`__.
