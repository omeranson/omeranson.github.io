---
layout: post
title: "Codewalk: Dragonflow's Publish-Subscribe Infrastructure - Part 3 - Driver implementation"
date: 2016-05-11
---

In this three-parter I will go over and annotate the code for Dragonflow's
publish/subscribe mechanism. In this third part, I will go over and annotate
two publish/subscribe driver implementations: ZMQ, and Redis. In [Part 1]({% post_url 2016-05-09-Codewalk-dragonflows-publish-subscribe-infrastructure-part-1%})
I introduced the API, and in [Part 2]({% post_url 2016-05-10-Codewalk-dragonflows-publish-subscribe-infrastructure-part-2%})
I went over code in Dragonflow which uses the API.

## Introduction

[In the first part]({% post_url 2016-05-09-Codewalk-dragonflows-publish-subscribe-infrastructure-part-1%}),
I went over the API of Dragonflow's publish/subscribe mechanism. 

[In the previous part]({% post_url 2016-05-10-Codewalk-dragonflows-publish-subscribe-infrastructure-part-2%}),
I went over two key locations which use the publish/subscribe API.

I'll now show two drivers that implement the above API.

1. ZMQ implements a
multi-process publish/subscribe mechanism, since it binds and listens to a port
on the controller.
2. Redis implements only a cross-node publish/subscribe mechanism, since the
Redis database also behaves as a broker.

## Implementation Examples

### ZMQ

ZMQ is the reference implementation. New drivers are encouraged to review this
implementation as an example and reference.

The ZMQ driver is a multi-process publisher. Therefore, the manager, publisher,
and subscriber have two implementation.

Unlike the previous section, I will review the ZMQ implementation top-down,
i.e. I'll first review the managers, then publisher abstract class and
implementations, and then the subscriber abstract class and implementations.

The original code is available [here](https://github.com/openstack/dragonflow/blob/b325a3a72edb78b1089b6d15265d857dca12867c/dragonflow/db/pubsub_drivers/zmq_pubsub_driver.py).

#### Managers

The first manager is the cross-node manager. Since ZMQ is a multi-proc pub/sub
mechanism, only the subscriber is loaded in NbApi, and the publisher is loaded
in the publisher service.

~~~ python
class ZMQPubSub(pub_sub_api.PubSubApi):
    def __init__(self):
        # Removed ...

    def get_publisher(self):
        return self.publisher

    def get_subscriber(self):
        return self.subscriber
~~~

The `get_publisher` and `get_subscriber` methods implement the API as promised.
They return the publisher and subscriber instances.

The `__init__` method, which is where the real magic happens, is:

~~~ python
    def __init__(self):
        super(ZMQPubSub, self).__init__()
        transport = cfg.CONF.df.publisher_transport
        if transport not in SUPPORTED_TRANSPORTS:
            message = _LE("zmq_pub_sub: Unsupported publisher_transport value "
                "%(transport)s, expected %(expected)s")
            LOG.error(message % {
                'transport': transport,
                'expected': str(SUPPORTED_TRANSPORTS)
            })
            raise UnsupportedTransportException(transport=transport)
        self.subscriber = ZMQSubscriberAgent()
        self.publisher = ZMQPublisherAgent()
~~~

Firstly, it calls the super-class' constructor. It then reads the transport
from the configuration. ZMQ supports (at least) TCP, UDP, and EPGM. The
code verifies that the transport is one of these. It raises an error otherwise.

This is the definition of SUPPORTED_TRANSPORTS:

~~~ python
SUPPORTED_TRANSPORTS = set(['tcp', 'epgm'])
~~~

Once the configuration is verified, the code creates a subscriber and publisher
instances, and stores them as members. They are later retrieved with
`get_publisher` and `get_subscriber` above.

The multi-process manager is very similar.

~~~ python
class ZMQPubSubMultiproc(pub_sub_api.PubSubApi):
    def __init__(self):
        super(ZMQPubSubMultiproc, self).__init__()
        self.subscriber = ZMQSubscriberMultiprocAgent()
        self.publisher = ZMQPublisherMultiprocAgent()

    def get_publisher(self):
        return self.publisher

    def get_subscriber(self):
        return self.subscriber
~~~

Since the communication is within
the node, and the protocol is defined by ZMQ (The protocol is `ipc`, and the
socket is a unix socket filename), no verification is needed.

The manager only has to instantiate the publisher and subscriber instances, and
save them as members. Note that these are different subscriber and publisher, specific for
inter-process communication.

#### Publisher

The inter-process and the inter-node publishers are very similar. Therefore,
there is an abstract class that does most of the work.

~~~ python
class ZMQPublisherAgentBase(pub_sub_api.PublisherApi):
    def __init__(self):
        self.socket = None
~~~

The abstract class' single member is the socket, through which the publisher
sends events.

~~~ python
    # Necessary, since it appears in the abstract class
    def initialize(self):
        super(ZMQPublisherAgentBase, self).initialize()
~~~

Even though this method does nothing, it is implemented so that inheriting
classes will not have to define it. The method itself must be defined, since it
appears in the abstract class `PublisherApi`.

~~~ python
    def send_event(self, update, topic=None):
        #NOTE(gampel) In this reference implementation we develop a trigger
        #based pub sub without sending the value mainly in order to avoid
        #consistency issues in th cost of extra latency i.e get
        if update.action != 'log':
            update.value = None

        if topic:
            update.topic = topic
        elif update.topic:
            topic = update.topic.encode('utf-8')
        else:
            topic = db_common.SEND_ALL_TOPIC
            update.topic = topic
        json_data = jsonutils.dumps(update.to_dict())
        data = pub_sub_api.pack_message(json_data)
        self.socket.send_multipart([topic, data])
        LOG.debug("sending %s" % update)
~~~

The main task of the publisher is the `send_event` method. Given that the
instance is already initialised, and the `socket` member is already set, the
`send_event` method in this abstract class sends the event. It is the
inheriting class' responsibility to make sure `socket` is set-up correctly.

I'll break down this method a bit more:

~~~ python
        #NOTE(gampel) In this reference implementation we develop a trigger
        #based pub sub without sending the value mainly in order to avoid
        #consistency issues in th cost of extra latency i.e get
        if update.action != 'log':
            update.value = None
~~~

As I said in [Part 1]({% post_url 2016-05-09-Codewalk-dragonflows-publish-subscribe-infrastructure-part-1%}),
in this implementation, the `value` member of the
update is always set to `None`. This is planned to be changed once
database consistency and reliability are implemented.

~~~ python
        if topic:
            update.topic = topic
        elif update.topic:
            topic = update.topic.encode('utf-8')
        else:
            topic = db_common.SEND_ALL_TOPIC
            update.topic = topic
~~~

This code sets up the correct topic, or channel. The default topic is
`db_common.SEND_ALL_TOPIC`. However, a specific topic can be set on the update
instance. That topic can be overriden by the `topic` argument to the 
`send_event` method.

The block of code above implements this behaviour. It also sanitizes the topic
to be encoded such that there will be no 'special characters'.

~~~ python
        json_data = jsonutils.dumps(update.to_dict())
        data = pub_sub_api.pack_message(json_data)
        self.socket.send_multipart([topic, data])
        LOG.debug("sending %s" % update)
~~~

Lastly, the method encodes the data to a JSON string, and sends it to the
socket. ZMQ's API defines that when a message is sent on a specific channel,
then the message is sent as a `(channel, data)` tuple.

Additionally, there is a debug log for the sent message.

The `pack_message` method serializes the data. It appears there is a bit of
double-serialisation here, as `jsonutils.dumps` serialises the data to a json
string.

Now we can review the two publisher implementations.

The `ZMQPublisherAgent` class implements the cross-node publisher. It is also
defined [here](https://github.com/openstack/dragonflow/blob/b325a3a72edb78b1089b6d15265d857dca12867c/dragonflow/db/pubsub_drivers/zmq_pubsub_driver.py).

~~~ python
class ZMQPublisherAgent(ZMQPublisherAgentBase):
    def __init__(self):
        super(ZMQPublisherAgent, self).__init__()
        self._endpoint = "{}://{}:{}".format(
            cfg.CONF.df.publisher_transport,
            cfg.CONF.df.publisher_bind_address,
            cfg.CONF.df.publisher_port,
        )

    def initialize(self):
        super(ZMQPublisherAgent, self).initialize()
        self._connect()

    def _connect(self):
        context = zmq.Context()
        self.socket = context.socket(zmq.PUB)
        LOG.debug("about to bind to network socket: %s" % self._endpoint)
        self.socket.bind(self._endpoint)
~~~

Firstly, it inherits the abstract class `ZMQPublisherAgentBase`.

The `__init__` (constructor) method reads the address and port to bind. It
includes the transport, address, and port. e.g. if the transport is `tcp`, port
`1234`, and any IP address, the resulting endpoint will be `tcp://*:1234`.

The `initialize` method connects the socket, i.e. it calls the `_connect`
method. The `_connect` method creates a ZMQ context and a socket of type
publisher. Then it binds the socket and listens to the address and port defined
in the constructor. The socket is set to the `socket` member.

The `ZMQPublisherMultiprocAgent` class implements the inter-process publisher.
It is also defined [here](https://github.com/openstack/dragonflow/blob/b325a3a72edb78b1089b6d15265d857dca12867c/dragonflow/db/pubsub_drivers/zmq_pubsub_driver.py).

~~~ python
class ZMQPublisherMultiprocAgent(ZMQPublisherAgentBase):
    def __init__(self):
        super(ZMQPublisherMultiprocAgent, self).__init__()
        self.ipc_socket = cfg.CONF.df.publisher_multiproc_socket

    def _connect(self):
        context = zmq.Context()
        self.socket = context.socket(zmq.PUSH)
        LOG.debug("about to connect to IPC socket: %s" % self.ipc_socket)
        self.socket.connect('ipc://%s' % self.ipc_socket)

    def send_event(self, update, topic=None):
        if not self.socket:
            self._connect()
        super(ZMQPublisherMultiprocAgent, self).send_event(update, topic)
~~~

It also inherits the abstract class `ZMQPublisherAgentBase`. However, in this
class, the connection is done using the lazy pattern. Therefore, the
`initialize` method is not defined, and inherits the empty implementation from
the base class.

The `__init__` (constructor) method simply reads the socket file from
configuration. This is a UNIX socket filename, over which the communication
between processes will be sent.

The `_connect` method connects to the multi-process subscriber. It creates a
ZMQ context, and a `PUSH` socket. The `socket` member is set to hold the ZMQ
socket. It then connects using the UNIX socket read from configuration. When
there are multiple writers (e.g. a Neutron service per CPU core) and a single
reader (e.g. the publisher service), the communication is of type PUSH/PULL. 

The `send_event` method sends the update via the base class. However, it first
verifies that the publisher is connected, and connects (by calling `_connect`)
if it isn't.

#### Subscriber

Like the publisher, the subscriber has a cross-node implementation and an
inter-process implementation. Both extend the base class
`ZMQSubscriberAgentBase`. All three are also available [here](https://github.com/openstack/dragonflow/blob/b325a3a72edb78b1089b6d15265d857dca12867c/dragonflow/db/pubsub_drivers/zmq_pubsub_driver.py).

~~~ python
class ZMQSubscriberAgentBase(pub_sub_api.SubscriberAgentBase):

    def __init__(self):
        super(ZMQSubscriberAgentBase, self).__init__()
        self.sub_socket = None

    def register_listen_address(self, uri):
        super(ZMQSubscriberAgentBase, self).register_listen_address(uri)
        #TODO(gampel)interrupt the sub socket recv and reconnect

    def connect(self):
        """Connect to the publisher"""

    def unregister_listen_address(self, uri):
        super(ZMQSubscriberAgentBase, self).unregister_listen_address(uri)
        #TODO(gampel)interrupt the sub socket recv and reconnect

    def register_topic(self, topic):
        super(ZMQSubscriberAgentBase, self).register_topic(topic)
        topic = topic.encode('ascii', 'ignore')
        if self.sub_socket:
            self.sub_socket.setsockopt(zmq.SUBSCRIBE, topic)

    def unregister_topic(self, topic):
        super(ZMQSubscriberAgentBase, self).unregister_topic(topic)
        topic = topic.encode('ascii', 'ignore')
        if self.sub_socket:
            self.sub_socket.setsockopt(zmq.UNSUBSCRIBE, topic)

    def run(self):
        self.sub_socket = self.connect()
        LOG.info(_LI("Starting  Subscriber on ports %(endpoints)s ")
                % {'endpoints': str(self.uri_list)})
        while True:
            try:
                eventlet.sleep(0)
                [topic, data] = self.sub_socket.recv_multipart()
                entry_json = pub_sub_api.unpack_message(data)
                message = jsonutils.loads(entry_json)
                self.db_changes_callback(
                    message['table'],
                    message['key'],
                    message['action'],
                    message['value'],
                    message['topic'],
                )
            except Exception as e:
                LOG.warning(e)
                self.sub_socket.close()
                self.sub_socket = self.connect()
                self.db_changes_callback(None, None, 'sync', None, None)
~~~

To make things easier, I'll review this class in bite-sized chunks as well.

Definition:

~~~ python
class ZMQSubscriberAgentBase(pub_sub_api.SubscriberAgentBase):
~~~

As expected, the base class extends the `SubscriberAgentBase` abstract class we
saw above, which implements the `SubscriberApi`.

~~~ python
    def __init__(self):
        super(ZMQSubscriberAgentBase, self).__init__()
        self.sub_socket = None

    def connect(self):
        """Connect to the publisher"""
~~~

The `__init__` (constructor) method calls the parent method, and initialises
the `sub_socket` member to `None`. This member will hold the subscriber's ZMQ
connection, from which it reads events.

The method `connect` is declared but not implemented. Subclasses are expected
to implement this method to initialise the `sub_socket` member.

~~~ python
    def register_listen_address(self, uri):
        super(ZMQSubscriberAgentBase, self).register_listen_address(uri)
        #TODO(gampel)interrupt the sub socket recv and reconnect

    def unregister_listen_address(self, uri):
        super(ZMQSubscriberAgentBase, self).unregister_listen_address(uri)
        #TODO(gampel)interrupt the sub socket recv and reconnect
~~~

These methods do not add funcitonality over their overriden counterparts. When
a URI is added or removed, the relevant connection should be made or destroyed.
This isn't the case yet, and the comments in the code are there to remind that
it needs to be added.

~~~ python
    def register_topic(self, topic):
        super(ZMQSubscriberAgentBase, self).register_topic(topic)
        topic = topic.encode('ascii', 'ignore')
        if self.sub_socket:
            self.sub_socket.setsockopt(zmq.SUBSCRIBE, topic)

    def unregister_topic(self, topic):
        super(ZMQSubscriberAgentBase, self).unregister_topic(topic)
        topic = topic.encode('ascii', 'ignore')
        if self.sub_socket:
            self.sub_socket.setsockopt(zmq.UNSUBSCRIBE, topic)
~~~

In addition to calling the base methods which add and remove the topic to the
topic list, the `register_topic` and `unregister_topic` also subscribe and
unsubscribe to the given topic (or channel) on the ZMQ socket, if it is already
connected (i.e. if it already exists).

~~~ python
    def run(self):
        self.sub_socket = self.connect()
        LOG.info(_LI("Starting  Subscriber on ports %(endpoints)s ")
                % {'endpoints': str(self.uri_list)})
        while True:
            try:
                eventlet.sleep(0)
                [topic, data] = self.sub_socket.recv_multipart()
                entry_json = pub_sub_api.unpack_message(data)
                message = jsonutils.loads(entry_json)
                self.db_changes_callback(
                    message['table'],
                    message['key'],
                    message['action'],
                    message['value'],
                    message['topic'],
                )
            except Exception as e:
                LOG.warning(e)
                self.sub_socket.close()
                self.sub_socket = self.connect()
                self.db_changes_callback(None, None, 'sync',
~~~

The `run` method connects the subscriber to the publishers. It then repeatedly
waits until an event is received by ZMQ. It parses the events, and passes it to
the callback. The callback was given to the `initialize` method, defined on the
base class.

In case of error, the connection is restarted by calling `sub_socket.close()`
and `connect`, and a re-sync event is sent via the callback.

Recall that this method is run in a separate thread. This is an important
implementation detail, which allows us to wait in a `While True:` loop.

There are two extensions to this class. The cross-node subscriber, and
inter-process subscriber. Both only implement the `connect` method, declared
in the base class `ZMQSubscriberAgentBase`.

The cross-node subscriber is implemented by `ZMQSubscriberAgent`.

~~~ python
class ZMQSubscriberAgent(ZMQSubscriberAgentBase):
    def connect(self):
        context = zmq.Context()
        socket = context.socket(zmq.SUB)
        for uri in self.uri_list:
            #TODO(gampel) handle exp zmq.EINVAL,zmq.EPROTONOSUPPORT
            LOG.debug("about to connect to network publisher at %s" % uri)
            socket.connect(uri)
        for topic in self.topic_list:
            socket.setsockopt(zmq.SUBSCRIBE, topic)
        return socket
~~~

`ZMQSubscriberAgent` implements the `connect` method which creates a ZMQ
context, and a subscriber socket (as opposed to publisher, push, or pull).
It then connects to every URI in the URI list, and registers to every topic (or
channel) in the topic list.

Recall that the URI and topic lists are maintained by the base class
`SubscriberAgentBase`.

The inter-process subscriber `ZMQSubscriberMultiprocAgent` is very similar.

~~~ python
class ZMQSubscriberMultiprocAgent(ZMQSubscriberAgentBase):
    def connect(self):
        context = zmq.Context()
        inproc_server = context.socket(zmq.PULL)
        ipc_socket = cfg.CONF.df.publisher_multiproc_socket
        LOG.debug("about to bind to IPC socket: %s" % ipc_socket)
        inproc_server.bind('ipc://%s' % ipc_socket)
        return inproc_server
~~~

In `ZMQSubscriberMultiprocAgent`'s implementation of `connect`, it also creates
a ZMQ context, and a socket of type PULL (Since there is one subscriber, but
many publishers; i.e. a publisher per CPU core). The connection address is the
IPC socket (the UNIX socket filename), to which it binds and waits for
publishers to connect.

### Redis

The Redis implementation is slightly simpler, since Redis is a broker based
publish/subscribe mechanism, with the database as a broker. This means that
inter-process (multiproc) implementations are not needed, and that the broker
manages some of the bookkeeping.

The Redis' driver implementation is available [here](https://github.com/openstack/dragonflow/blob/b325a3a72edb78b1089b6d15265d857dca12867c/dragonflow/db/pubsub_drivers/redis_db_pubsub_driver.py).

#### Manager

The Redis' driver manager is very similar to ZMQ's:

~~~ python
class RedisPubSub(pub_sub_api.PubSubApi):

    def __init__(self):
        super(RedisPubSub, self).__init__()
        self.subscriber = RedisSubscriberAgent()
        self.publisher = RedisPublisherAgent()
        self.redis_mgt = None

    def get_publisher(self):
        return self.publisher

    def get_subscriber(self):
        return self.subscriber
~~~

It also creates a publisher and subscriber instances in the `__init__`
(constructor) method, and stores them to the relevant members. The
`get_publisher` and `get_subscriber` also behave as simple getters to these
members.

#### Publisher

The Redis' driver publisher is implemented in `RedisPublisherAgent`.

~~~ python
class RedisPublisherAgent(pub_sub_api.PublisherApi):

    def __init__(self):
        super(RedisPublisherAgent, self).__init__()
        self.remote = None
        self.client = None
        self.redis_mgt = None

    def initialize(self):
        # find a publisher server node
        super(RedisPublisherAgent, self).initialize()
        self.redis_mgt = RedisMgt.get_instance(cfg.CONF.df.remote_db_ip,
                                              cfg.CONF.df.remote_db_port)
        self.remote = self.redis_mgt.pubsub_select_node_idx()
        ip_port = self.remote.split(':')
        self.client = redis.client.StrictRedis(host=ip_port[0],
                                               port=ip_port[1])

    def send_event(self, update, topic=None):
        if topic:
            update.topic = topic
        local_topic = update.topic
        event_json = jsonutils.dumps(update.to_dict())
        local_topic = local_topic.encode('utf8')
        data = pub_sub_api.pack_message(event_json)
        self.client.publish(local_topic, data)
~~~

The `__init__` constructor method only initialises the members to `None`. The
members are:

* `remote`
: The IP and port of the database (and broker). The
* `client`
: An instance of `StrictRedis`, a python implementation to a Redis database
  client.
* `redis_mgt`
: A `RedisMgt` instance, which is a management object, responsible for the
  bookkeeping of the connection to a Redis database. I will cover it at another
  time.

The `initialize` method connects to the Redis database (or broker). It gets
the `RedisMgt` instance, and queries it for a node to use for
Publish/Subscribe. The node is returned as an 'IP:port' string and has to be
split to IP and port (the 0th and 1st elements in `ip_port`). Then a
`StrictRedis` instance is created to connect to the given node, and saved as
the `client` member.

The `send_event` method sends the given update. Similar to ZMQ's `send_event`,
it allows overriding the topic, then encodes it as a JSON, and packs the
message. The resulting message is published using `StrictRedis` API, which
will eventually call Redis' PUBLISH command.

#### Subscriber

The subscriber is implemented in `RedisSubscriberAgent`. The implementation is
fairly straight-forwards, especially after we have seen the ZMQ subscriber
implementation. However, it is a bit long, so I will go over it in chunks.

~~~ python
class RedisSubscriberAgent(pub_sub_api.SubscriberAgentBase):

    def __init__(self):
        super(RedisSubscriberAgent, self).__init__()
        self.remote = []
        self.client = None
        self.ip = ""
        self.plugin_updates_port = ""
        self.pub_sub = None
        self.redis_mgt = None
~~~

As expected, the class inherits the `SubscriberAgentBase` base class.

The `__init__` (constructor) method initialises the following members:

* `remote`
: This member is a string, which will hold the broker's IP address
  and port number.
* `client`
: An instance of `StrictRedis`, the Redis python client. Published
  events will be read via this member.
* `ip`
: The IP address part of `remote`. i.e. the IP address of the database
  (or broker).
* `plugin_updates_port`
: The port number part of `remote`. i.e. the port the
  database (or broker) is listening on.
* `pub_sub`
: The Redis python client (`StrictRedis`) API for publish/subscribe
  operations.
* `redis_mgt`
: A `RedisMgt` instance, the management object responsible for
  the bookkeeping of the connection to the Redis database.

~~~ python
    def initialize(self, callback):
        # find a subscriber server node and run daemon
        super(RedisSubscriberAgent, self).initialize(callback)
        self.redis_mgt = RedisMgt.get_instance(cfg.CONF.df.remote_db_ip,
                                              cfg.CONF.df.remote_db_port)
        self.remote = self.redis_mgt.pubsub_select_node_idx()
        ip_port = self.remote.split(':')
        self.client = \
            redis.client.StrictRedis(host=ip_port[0], port=ip_port[1])
        self.ip = ip_port[0]
        self.plugin_updates_port = ip_port[1]
        self.pub_sub = self.client.pubsub()
~~~

The `initialize` method connects to the Redis database (or broker), and
initialises the `pub_sub` member.

It begins by setting the `redis_mgt` member to an instance of `RedisMgt`. It
then queries that instance for a publish/subscribe node to which it can
connect. The result is an 'IP:port' string, which is split to an IP address and
port number.

The `client` member is then initialised to a `StrictRedis` instance connecting
to the given node. Additionally, the `ip` and `plugin_updates_port` members
are set to the IP and port of the node.

Lastly, the `pub_sub` member is set to the `pubsub` element in the Redis
client.

~~~ python
    def register_listen_address(self, uri):
        super(RedisSubscriberAgent, self).register_listen_address(uri)

    def unregister_listen_address(self, uri):
        super(RedisSubscriberAgent, self).unregister_listen_address(uri)

    def register_topic(self, topic):
        self.pub_sub.subscribe(topic)

    def unregister_topic(self, topic):
        self.pub_sub.unsubscribe(topic)
~~~

In the Redis implementation, adding and removing a URI has no real meaning,
since all Pub/Sub communication is done with the broker and the assigned node.
Therefore, in this implementation, the URIs are only passed to the abstract
class. In fact, it looks like they can be removed.

Redis' pub/sub code maintains the topics (or channels) to which the subscriber
listens. Therefore, when registering to or unregistering from a topic,
`register_topic` and `unregister_topic` simply pass the command to Redis'
pub/sub driver.

~~~ python
    def run(self):
        while True:
            eventlet.sleep(0)
            try:
                for data in self.pub_sub.listen():
                    if 'subscribe' == data['type']:
                        continue
                    if 'unsubscribe' == data['type']:
                        continue
                    if 'message' == data['type']:
                        entry = pub_sub_api.unpack_message(data['data'])
                        entry_json = jsonutils.loads(entry)
                        self.db_changes_callback(
                            entry_json['table'],
                            entry_json['key'],
                            entry_json['action'],
                            entry_json['value'],
                            entry_json['topic'])

            except Exception as e:
                LOG.warning(e)
                try:
                    connection = self.pub_sub.connection
                    connection.connect()
                    self.db_changes_callback(None, None, 'sync', None, None)
                except Exception as e:
                    LOG.exception(_LE("reconnect error %(ip)s:%(port)s")
                                  % {'ip': self.ip,
                                     'port': self.plugin_updates_port})
~~~

In the Redis driver, `run` also has an infinite while loop, that conitnuously
listens to events from Redis' pub/sub code. If the event type is 'subscribe' or
'unsubscribe', these are subscription events that can be ignored.

A real event arrives when the event type is `message`. In which case, the event
data is the json string we constructed in the publisher's `send_event`. We
extract it here, unpack it, and parse it to a python dictionary. We then call
the callback method `db_changes_callback` (which was provided in the call to
`initialize`) with the parameters from the event.

In case an error occurs, either in the Redis code, or in the callback code, we
issue a warning, and try to reconnect to the node. Additionally, we send a
re-sync event to the callback method, in case any database update events were
lost.

## Conclusion

In this part we have seen two implementations of the publish/subscribe API,
namely ZMQ, and Redis.

In this three parter I went over the publish/subscribe API, its usage, and two
implementations of it (ZMQ, and Redis).

