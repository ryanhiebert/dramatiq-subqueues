Designing Subqueues for Dramatiq
++++++++++++++++++++++++++++++++

It appears that there is no way to have consumers register
to consume from queues defined as a wildcard in RabbitMQ.
I do want to make sure that the subqueues feature uses RabbitMQ,
so some cleverness is gonna be required.

In the standard RabbitMQ broker implementation,
Dramatiq creates two queues per declared queue.
For example, if the name of the queue is ``queue``,
then there will be two RabbitMQ queues created:
``queue`` and ``queue.DQ`` (for the delayed messages).

We need two things in order to get subqueues working.
First, we need a way to correlate subqueues with queues,
so that the consumer can know all of the subqueues that
it should be listening on.

Second, we need a way to signal to active consumers
that new subqueues have been created and listened to,
and that outdated subqueues are no longer active.

To make long-term storage of subqueue registration to the main queue,
my thought is to parallel RabbitMQ queue called ``queue.SQ``,
analagous to ``queue.DQ``, with the ``SQ`` standing for "sub queue".
In this will be a message for each registered or deregistered subqueue.
Upon startup, a new worker will read from this queue exactly once
to know what subqueues it should be looking for,
then ensure that all messages are returned to the queue.
No further processing of this queue will take place
during the lifetime of the consumer.

When new subqueues are created or abandoned,
a new message indicating the change in subqueues
should be sent to the ``queue.SQ`` queue.
This will inform all consumers that start after this point
on which subqueues are part of this queue.

To enable active consumers to track these changes to subqueues,
Each consumer should also specify a queue with a consumer specific name,
that will be registered to a fan-out exchange of ``queue.SQ``,
or ``queue.SQS`` if the name conflicts,
as well as the ``queue.SQ`` queue itself.
When new subqueues are created or abandoned,
the same message added to the persistent ``queue.SQ``
will also be sent to this exchange,
putting the message in the queue for each worker to process.
These messages should *not* be re-enqueued, since they
are meant as notifications, not perisistent storage.
The queues that are created dynamically for each worker
might look something like ``consumer.<uuid>``,
where ``<uuid>`` is a random uuid string created by the consumer on startup.
These queues should be automatically deleted on worker disconnect or timeout.
Each of these queues may be hooked up to receive notifications
for multiple subqueue exchanges, so each message should contain
a reference to the queue to which the subqueue is registered.
