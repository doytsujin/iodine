# Ruby Pub/Sub API Specification Draft

## Purpose

The pub/sub design is idiomatic to WebSocket and EventSource approaches as well as other reactive programming techniques.

The purpose of this specification is to offer a recommendation for pub/sub design that will allow applications to be implementation agnostic (not care which pub/sub extension is used)\*.

Simply put, applications will not have to worry about the chosen pub/sub implementation or about inter-process communication.

This should simplify the idiomatic `subscribe` / `publish` approach to real-time data pushing.

\* The pub/sub extension could be implemented by any external library as long as the API conforms to the specification. The extension will have to manage the fact that some servers `fork` and manage inter-process communication for pub/sub propagation (or limit it's support to specific servers). Also, servers that opt to implement the pub/sub layer, could perform optimizations related to connection handling and pub/sub lifetimes.

## Pub/Sub handling

Conforming Pub/Sub implementations **MUST** implement the following pub/sub related methods within the WebSocket/SSE `client` object (as defined in the Rack WebSockets / EventSource specification draft):

* `subscribe(to, opt = {}) { |from, message| optional_block }` where `to` is a named *channel* and `opt` is a Hash object that *SHOULD* support the following possible keys (unsupported keys *MUST* be ignored):

    * `:match` indicates a matching algorithm should be applied to the `to` variable (`to` is a pattern).
    
        Possible values should include [`:redis`](https://github.com/antirez/redis/blob/398b2084af067ae4d669e0ce5a63d3bc89c639d3/src/util.c#L46-L167), [`:nats`](https://nats.io/documentation/faq/#wildcards) or [`:rabbitmq`](https://www.rabbitmq.com/tutorials/tutorial-five-ruby.html). Pub/Sub implementations *MAY* support none, some or all of these common pattern resolution schemes.
    
    * `:handler` is an alternative to the optional block. It should accept Proc like objects (objects that answer to `.call(from, msg)`).

    * If an optional `block` (or `:handler`) is provided, if will be called when a publication was received. Otherwise, the message alone (**not** the channel data) **MUST** be sent directly to the WebSocket / EventSource client.

    * `:as` accepts either `:text` or `:binary` Symbol objects.

        This option is only valid if the optional `block` is missing and the connection is a WebSocket connection. Note that SSE connections are limited to text data by design.

        This will dictate the encoding for outgoing WebSocket message when publications are directly sent to the client (as a text message or a binary blob). `:text` will be the default value for a missing `:as` option.
    
    If a subscription to `to` already exists, it should be *replaced* by the new subscription (the old subscription should be canceled / unsubscribed).

    When the `subscribe` method is called within a WebSocket / SSE Callback object, the subscription must be closed automatically when the connection is closed.
    
    The `subscribe` method **MUST** return `nil` on a known failure (i.e., when the connection is already closed), or any truthful value on success.

    A global variation for this method (allowing global subscriptions to be created) **SHOULD** be made available as `Rack::PubSub.subscribe`.

    When the `subscribe` method isn't called from within a connection, it should be considered a global (non connection related) subscription and an exception should be raised if a `block` or `:handler` isn't provided by the user.

* `unsubscribe(from)` should cancel a subscription to the `from` named channel / pattern.

* `publish(to, message, engine = nil)` (preferably supporting named arguments) where:

    * `to` a String that identifies the channel / stream / subject for the publication ("channel" is the semantic used by Redis, it is similar to "subject" or "stream" in other pub/sub systems).

    * `message` a String with containing the data to be published.

    * `engine` (optional) routed the publish method to the specified Pub/Sub Engine (see later on). If none is specified, the default engine should be used.

    The `publish` method must return `true` if a publication was scheduled (not necessarily performed). If it's already known that the publication would fail, the method should return `false`.

    An implementation **MUST** call the relevant PubSubEngine's `publish` method after performing any internal book keeping logic. If `engine` is `nil`, the default PubSubEngine should be called. If `engine` is `false`, the implementation **MUST** forward the published message to the actual clients (if any).

    A global alias for this method (allowing it to be accessed from outside active connections) should be defined as `Rack::PubSub.publish`.

Implementations **MUST** implement the following methods:

* `Rack::PubSub.register(engine)` where `engine` is a PubSubEngine object as described in this specification.

    When a pub/sub engine is registered, the implementation **MUST** inform the engine of any existing or future subscriptions.

    The implementation **MUST** call the engine's `subscribe` callback for each existing (and future) subscription.

* `Rack::PubSub.deregister(engine)` where `engine` is a PubSubEngine object as described in this specification.

    Removes an engine from the pub/sub registration. The opposit of 

* `Rack::PubSub.default_engine = engine` sets a default pub/sub engine, where `engine` is a PubSubEngine object as described in this specification.

    Implementations **MUST** forward any `publish` method calls to the default pub/sub engine.

* `Rack::PubSub.default_engine` returns the current default pub/sub engine, where the engine is a PubSubEngine object as described in this specification.

* `Rack::PubSub.reset(engine)` where `engine` is a PubSubEngine object as described in this specification.

    Implementations **MUST** behave as if the engine was newly registered and (re)inform the engine of any existing subscriptions by calling engine's `subscribe` callback for each existing subscription.

Implementations **MAY** implement pub/sub internally (in which case the `pubsub_default` engine is the server itself or a server's module).

However, servers **MUST** support external pub/sub "engines" as described above, using PubSubEngine objects.

PubSubEngine objects **MUST** implement the following methods:

* `subscribe(channel, as=nil)` this method performs the subscription to the specified channel.

    If `as` is a Symbol that the engine recognizes (i.e., `:redis`, `:nats`, etc'), the engine should behave accordingly. i.e., the value `:redis` on a Redis engine will invoke the PSUBSCRIBE Redis command.

    The method must return `true` if a subscription was scheduled (or performed) or `false` if the subscription is known to fail.

    This method will be called by the server (for each registered engine). The engine may assume that the method would never be called directly by an application.

* `unsubscribe(channel, as=nil)` this method performs closes the subscription to the specified channel.

    The method's semantics are similar to `subscribe`.

    This method will be called by the server (for each registered engine). The engine may assume that the method would never be called directly by an application.

* `publish(channel, message)` where both `channel` and `message` are String object.

    This method will be called by the server when a message is published using the engine.

    The engine **MUST** assume that the method might called directly by an application.

When a PubSubEngine object receives a published message, it should call:

```ruby
Rack::PubSub.publish channel: channel, message: message, engine: false
```
