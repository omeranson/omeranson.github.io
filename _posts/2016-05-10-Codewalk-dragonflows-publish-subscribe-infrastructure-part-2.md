---
layout: post
title: "Codewalk: Dragonflow's Publish-Subscribe Infrastructure - Part 2 - Usage"
date: 2016-05-10
---

In this three-parter I will go over and annotate the code for Dragonflow's
publish/subscribe mechanism. In this second part, I will go over and annotate
two key locations where the API is used. Namely the `NbApi` class, and the
publisher service. In [Part 1]({% post_url 2016-05-09-Codewalk-dragonflows-publish-subscribe-infrastructure-part-1%})
I introduced the API, and in [Part 3]({% post_url 2016-05-11-Codewalk-dragonflows-publish-subscribe-infrastructure-part-3%})
I will show driver implementation examples.

## Introduction

[In the previous part]({% post_url 2016-05-09-Codewalk-dragonflows-publish-subscribe-infrastructure-part-1%}),
I went over the API of Dragonflow's publish/subscribe mechanism. In this part,
I will go over two key locations where this API is used.

The `NbApi`, which will hopefully be covered in another post, is the
abstraction layer used to access the database. For instance, database changes
are done via this class, when Dragonflow is run as a Neutron service plugin.

The publisher service is an external process that is currently used to proxy
the published events, in cases where the publishers cannot do so directly. This
happens when the publisher need complete control over the port to which it
binds, e.g. when using TCP.

## Usage

### NbApi

The code for `NbApi` is available [here](https://github.com/openstack/dragonflow/blob/b325a3a72edb78b1089b6d15265d857dca12867c/dragonflow/db/api_nb.py).
I will not go over the entire class now. However, I will show some usage of the
Pub/Sub API. Out of context, this may seem a bit unclear. I will (hopefully) go
over this again in a post dedicated to `NbApi`.

As a first step, the `NbApi` needs to get an instance of the publisher and the
subscriber. This is done in `_get_publisher` and `_get_subscriber` methods.

~~~ python
    def _get_publisher(self):
        if cfg.CONF.df.pub_sub_use_multiproc:
            pubsub_driver_name = cfg.CONF.df.pub_sub_multiproc_driver
        else:
            pubsub_driver_name = cfg.CONF.df.pub_sub_driver
        pub_sub_driver = df_utils.load_driver(
                                    pubsub_driver_name,
                                    df_utils.DF_PUBSUB_DRIVER_NAMESPACE)
        return pub_sub_driver.get_publisher()
~~~

In the `_get_publisher` case, the publisher may be a cross-node publisher (in
case of a broker based pub/sub mechanism, e.g. Redis) or an inter-process
publisher (e.g. ZMQ). The code checks the configuration to see which publisher
is needed, and then loads and constructs (using `df_utils.load_driver`) the
class of the correct manager. It then returns the manager's publisher instance.

~~~ python
    def _get_subscriber(self):
        pub_sub_driver = df_utils.load_driver(
                                    cfg.CONF.df.pub_sub_driver,
                                    df_utils.DF_PUBSUB_DRIVER_NAMESPACE)
        return pub_sub_driver.get_subscriber()
~~~

The `_get_subscriber` case is simpler, since the cross-node subscriber is
always used. Therefore, the code loads and constructs (again, using
`df_utils.load_driver`) the cross-node pub/sub manager, and returns the
subscriber instance the manager returns.

I would like to expand on the configuration options:
* `pub_sub_driver`
: This is the name of the class of the cross-node publish/subscribe manager.
* `pub_sub_multiproc_driver`
: This is the name of the class of the inter-process publish/subscribe manager.

These values are mapped in the [setup.cfg](https://github.com/openstack/dragonflow/blob/b325a3a72edb78b1089b6d15265d857dca12867c/setup.cfg)
file, in the `dragonflow.pubsub_driver` section. This is the value of
`df_utils.DF_PUBSUB_DRIVER_NAMESPACE`.

For example, the following is the configuration used for ZMQ

~~~
[df]
# ... More configuration
pub_sub_driver = zmq_pubsub_driver
pub_sub_multiproc_driver = zmq_pubsub_multiproc_driver
~~~

Once the publisher and subscriber are created, they need to be started. The
publisher is started simply with a call to `self.publisher.initialize()`. The
subscriber is started with a call to the method `_start_subscriber`.

~~~ python
    def _start_subsciber(self):
        self.subscriber.initialize(self.db_change_callback)
        # TODO(oanson) Move publishers_ips to DF DB.
        publishers_ips = cfg.CONF.df.publishers_ips
        for ip in publishers_ips:
            uri = '%s://%s:%s' % (
                cfg.CONF.df.publisher_transport,
                ip,
                cfg.CONF.df.publisher_port
            )
            self.subscriber.register_listen_address(uri)
        self.subscriber.daemonize()
~~~

The `_start_subscriber` starts by initialising the subscriber with a call to
`initialize`, passing `self.db_change_callback` as the callback. Then the code
reads all the publishers from configuration (In future also from the
distributed database) and passes their URI to `register_listen_address`.
Lastly, the code calls `daemonize`, which starts a new thread, and calls the
subscriber's `run` method within it, to continuously wait for events.

This is the `db_change_callback` method.

~~~ python
    def db_change_callback(self, table, key, action, value, topic=None):
        update = DbUpdate(table, key, action, value, topic=topic)
        LOG.info(_LI("Pushing Update to Queue: %s"), update)
        self._queue.put(update)
        eventlet.sleep(0)
~~~

The `db_change_callback` receives all the arguments of an event (i.e. table,
key, action, value, and topic). It wraps these arguments in a `DbUpdate`
instance, and adds it to a queue (the member `_queue`) for processing. This
processing will be done later, by another thread.

Since we are using greenthreads (which mean cooperative threading), the call to
`eventlet.sleep` is necessary to allow that other thread to work. The call to
`db_change_callback` is done in the subscriber's thread, and we want the actual
event handling to be done in `NbApi`'s thread.

Once `NbApi` is ready to send an event via the pub/sub mechanism, it calls the
method `_send_db_change_event`.

~~~ python
    def _send_db_change_event(self, table, key, action, value, topic):
        if self.use_pubsub:
            if not self.enable_selective_topo_dist:
                topic = SEND_ALL_TOPIC
            update = DbUpdate(table, key, action, value, topic=topic)
            self.publisher.send_event(update)
            eventlet.sleep(0)
~~~

This method receives all the parameters of an event (table, key, action, value,
and topic). If the pub/sub mechanism is not used (denoted by the member
`use_pubsub`), then the method returns and does nothing. Otherwise, the method
continues.

If selective topology distribution feature is not used (denoted by the member
`enable_selective_topo_dist`), then the code overrides the topic with
the special topic `SEND_ALL_TOPIC`, which is the topic to which all subscribers
listen.

Finally, the update object is created with a call to `DbUpdate`, and sent via
the publisher by calling `self.publisher.send_event(update)`. Since we are
using greenthreads (which mean cooperative threading), the call to
`eventlet.sleep` allows other threads to work, if necessary. Specifically, the
`send_event` method may use a worker thread too.

### Publisher Service

Another location that heavily uses the Pub/Sub API is the Dragonflow publisher
service. This is the service that runs on the Controller Node, and binds the IP
and port to allow subscribers to connect.

The code for this module can be found [here](https://github.com/openstack/dragonflow/blob/b325a3a72edb78b1089b6d15265d857dca12867c/dragonflow/controller/df_publisher_service.py).

~~~ python
def main():
    cfg.CONF.register_opts(common_params.df_opts, 'df')
    common_config.init(sys.argv[1:])
    config.setup_logging()
    service = PublisherService()
    service.initialize()
    service.run()

~~~

Since this is a service, it runs as a separate process. This process starts at
the `main` function.

The `main` function registers the Dragonflow configuration options, and reads
the given configuration files with a call to `common_config.init`. It also sets
up the logging mechanism (by calling `config.setup_logging`).

Lastly, it creates an instance of `PublisherService`, initializes it, and calls
its main function.

To allow the python script to be called without specifying that we want the
`main` function in particular, the following code exists.

~~~ python
if __name__ == "__main__":
    main()
~~~

This code verifies that this python module is the main module, as opposed to
being imported from another module, and then calls the `main` function.

The main body of the publisher service is implemented in the `PublisherService`
class. It is a bit long, so again I separate it into chewable components.

~~~ python
class PublisherService(object):
    def __init__(self):
        self._queue = Queue()
        self.publisher = self._get_publisher()
        self.multiproc_subscriber = self._get_multiproc_subscriber()
        nb_driver_class = importutils.import_class(cfg.CONF.df.nb_db_class)
        self.db = nb_driver_class()
        self.uuid = pub_sub_api.generate_publisher_uuid()
        self._rate_limit = df_utils.RateLimiter(2, 30)
~~~

The `PublisherService` class doesn't inherit from any other class.

The `__init__` (constructor) method initialises the instance's members.
Specifically, these members are:

* `publisher`
: A publisher instance, through which the event will be sent to
  other nodes.

* `multiproc_subscriber`
: A subscriber instance, throught which events that
  have to be published will be received. These events are sent by the
  publishers used e.g. in `NbApi` as we saw above.

* `_queue`
: This queue is used to pass events from the `multiproc_subscriber`
  thread to the publisher service's main thread.

* `db`
: This is  the API that allows access to the database. Since the
  database is also a pluggable part of Dragonflow, the class name is read from
  configuration, and the class is imported into `nb_driver_class` using
  `importutils.import_class`. The instance is then constructed.

* `uuid`
: This is a unique identifier of this publisher service. The uuid is
  generated by a hash on the node's fully qualified domain name.

* `_rate_limit`
: An instance of `RateLimiter`, which is a utility class
  allowing callers to limit themselves to `n` actions (first parameter) in `m`
  seconds (second parameter).

As an aside, the `uuid` is generated by `generate_publisher_uuid`, available
[here](https://github.com/openstack/dragonflow/blob/b325a3a72edb78b1089b6d15265d857dca12867c/dragonflow/db/pub_sub_api.py).

~~~ python
def generate_publisher_uuid():
    """
    Generate a non-random uuid based on the fully qualified domain name.
    This UUID is supposed to remain the same across service restarts.
    """
    return str(uuid.uuid5(uuid.NAMESPACE_DNS, socket.getfqdn()))
~~~

It returns a textual representation (call to `str`) of the uuid, which is
generated by a SHA-1 hash (call to `uuid.uuid5`) on the DNS namespace
(Retrieved with `uuid.NAMESPACE_DNS`) and the node's fully qualified domain
name (Retrieved using `socket.getfqdn()`).

The publisher is constructed with a call to `_get_publisher`.

~~~ python
    def _get_publisher(self):
        pub_sub_driver = df_utils.load_driver(
                                    cfg.CONF.df.pub_sub_driver,
                                    df_utils.DF_PUBSUB_DRIVER_NAMESPACE)
        return pub_sub_driver.get_publisher()
~~~

The selected publisher is always the cross-node publisher. Therefore, we
retrieve an instnace of the cross-node pub/sub manager using
`df_utils.load_driver`, and return the publisher it creates.

Similarly, the subscriber is constructed with a call to
`_get_multiproc_subscriber`.

~~~ python
    def _get_multiproc_subscriber(self):
        """
        Return the subscriber for inter-process communication. If multi-proc
        communication is not use (i.e. disabled from config), return None.
        """
        if not cfg.CONF.df.pub_sub_use_multiproc:
            return None
        pub_sub_driver = df_utils.load_driver(
                                    cfg.CONF.df.pub_sub_multiproc_driver,
                                    df_utils.DF_PUBSUB_DRIVER_NAMESPACE)
        return pub_sub_driver.get_subscriber()
~~~

Firstly, this method verifies that inter-process publish/subscribe is used. We
will see later why the publisher service may be used without it.

If `pub_sub_use_multiproc` is false, i.e. inter-process publish/subscribe is
not used, the subscriber is `None`. Otherwise, the code retrieves an instance
of the inter-process pub/sub manager, and returns its subscriber.

~~~ python
    def initialize(self):
        if self.multiproc_subscriber:
            self.multiproc_subscriber.initialize(self._append_event_to_queue)
        self.publisher.initialize()
        # TODO(oanson) TableMonitor initialisation goes here
~~~

The `initialize` method initialises the subscriber (if it exists), and the
publisher.

In future, there are plans to add database table monitors to the publisher
service. A table monitor will regularly sample a database table, and if it has
changed, send an event. This is one of the reasons a publisher service may be
required, even if inter-process publish/subscribe is not used.

The callback function given to the subscriber is `_append_event_to_queue`.

~~~ python
    def _append_event_to_queue(self, table, key, action, value, topic):
        event = db_common.DbUpdate(table, key, action, value, topic=topic)
        self._queue.put(event)
        eventlet.sleep(0)
~~~

Very similarly to `NbApi`'s implementation, this method also wraps the event
arguments in a `DbUpdate` instance, and adds them to the queue. Recall that
this method runs on the subscriber's thread. It calls `eventlet.sleep` to allow
the publisher service's main thread to take over and process the event.

~~~ python
    def run(self):
        if self.multiproc_subscriber:
            self.multiproc_subscriber.daemonize()
        self.db.initialize(
            db_ip=cfg.CONF.df.remote_db_ip,
            db_port=cfg.CONF.df.remote_db_port,
            config=cfg.CONF.df
        )
        self._register_as_publisher()
        self._publishers_table_monitor = pub_sub_api.StalePublisherMonitor(
            self.db,
            self.publisher,
            cfg.CONF.df.publisher_timeout
        )
        self._publishers_table_monitor.daemonize()
        # TODO(oanson) TableMonitor daemonize will go here
        while True:
            try:
                event = self._queue.get()
                self.publisher.send_event(event)
                if event.table != pub_sub_api.PUBLISHER_TABLE:
                    self._update_timestamp_in_db()
                eventlet.sleep(0)
            except Exception as e:
                LOG.warning(_LW("Exception in main loop: {}, {}").format(
                    e, traceback.format_exc()
                ))
                # Ignore
~~~

The `run` method can be separated to two main parts - some more initialisation,
and the main loop (It may have been better to move the initialisation code to
`initialize`).

In the initialisation section, the code first creates the subscriber thread,
and starts its main loop, by calling the subscriber's `daemonize` method. This
is done only if the subscriber exists (which only happens when multi-proc
pub/sub is used).

It then initializes the member `db`, which connects to the database. The
database parameters, such as IP and port, are read from the configuration.

The publisher service registers itself as a publisher via
`_register_as_publisher`, which I will go over shortly.

It then creates a `StalePublisherMonitor` instance, which monitors the
publishers' table in the database, and removes publishers which have been idle
for too long. This instance also runs in a separate thread, so the call to
`daemonize` starts up that new thread, and calls the monitor's main method in
that thread.

Lastly, once other table monitors will be migrated to the publisher service,
their initialisation and daemonisation can be done in place of the comment.

In the main loop section is surrounded by a `while True` loop. It waits for an
event to appear in the queue by calling `_queue.get()`. This method blocks
until an element appears in `_queue`. Once an event arrives, either from the
subscriber thread or one of the table monitors, the publisher service forwards
it to the subcribers via the publisher's `send_event` method.

If the event did not originate from the publishers' table, the publisher's
timestamp is also updated with a call to `_update_timestamp_in_db`. For this
purpose, updates from the publishers' table are ignored. Otherwise, this would
cause a feedback loop, where a publishers' table update would update the
publisher's timestamp in the publishers' table.

Lastly, the other threads are given a chance to run by calling
`eventlet.sleep`.

In case an exception occurs, it is ignored. The publisher service tries to
continue working. It does emit a warning to the log, to facilitate debugging.

In the initialisation phase of `run`, there was a call to
`_register_as_publisher`.

~~~ python
    def _register_as_publisher(self):
        publisher = {
            'id': self.uuid,
            'uri': self._get_uri(),
            'last_activity_timestamp': time.time(),
        }
        publisher_json = jsonutils.dumps(publisher)
        self.db.create_key(
            pub_sub_api.PUBLISHER_TABLE,
            self.uuid, publisher_json
        )
~~~

This method adds a publisher record to the publishers' table in the database.
The publisher record includes:

* `id`
: the publisher's id, created during construction.
* `uri`
: The address that can be used to connect to the pulisher.
* `last_activity_timestamp`
: The last time that publisher had any activity.

This information is serialised to a json string, and added to the database with
the publisher's id as key.

The publisher's URI is retrieved with a call to `_get_uri`.

~~~ python
    def _get_uri(self):
        ip = cfg.CONF.df.publisher_bind_address
        if ip == '*' or ip == '127.0.0.1':
            ip = cfg.CONF.df.local_ip
        return "{}://{}:{}".format(
            cfg.CONF.df.publisher_transport,
            ip,
            cfg.CONF.df.publisher_port,
        )
~~~

The transport, IP, and port are taken from the configuration. The method then
concatenates the parameters to a URI. e.g. if the transport is 'tcp', the port
is 1234, and the IP is 192.168.17.105, then the result is
'tcp://192.168.17.105:1234'.

The IP address in the configuration may be '*', or '127.0.0.1'. This is to say
that the publisher can be connected to using any of the node's IP addresses.
Therefore, the node's IP address is read from configuration, and used instead.

Lastly, we saw that the publisher service updates its timestamp by calling
`_update_timestamp_in_db`.

~~~ python
    def _update_timestamp_in_db(self):
        if self._rate_limit():
            return
        try:
            publisher_json = self.db.get_key(
                pub_sub_api.PUBLISHER_TABLE,
                self.uuid,
            )
            publisher = jsonutils.loads(publisher_json)
            publisher['last_activity_timestamp'] = time.time()
            publisher_json = jsonutils.dumps(publisher)
            self.db.set_key(
                pub_sub_api.PUBLISHER_TABLE,
                self.uuid,
                publisher_json
            )
        except exceptions.DBKeyNotFound:
            self._register_as_publisher()
~~~

Firstly, this method verifies that it does not flood the deployment with
modifications in the publishers' table, by calling the rate limiter via the
`_rate_limit` member. This is important, since database modifications are
constly in and of themselves, and they generate events that other nodes have to
process.

The method then reads the existing publisher record from the database by its
ID with a call to `db.get_key`. It parses the JSON string to a python
dictionary using `jsonutils.loads`. It then updates the
`last_activity_timestamp` field with the current time (retrieved with
`time.time()`). It then serialises the dictionary back to a json string
(`jsonutils.dumps`) and updates the database (with `db.set_key`).

In case the publisher record is not found in the database, a `DBKeyNotFound`
exception is raised. In this case, the publisher creates the record anew in the
database.

## Conclusion

In this part, we have seen how the publish/subscribe mechanism is used.

[In the next part]({% post_url 2016-05-11-Codewalk-dragonflows-publish-subscribe-infrastructure-part-3%}),
I will show two driver implementations of it.

