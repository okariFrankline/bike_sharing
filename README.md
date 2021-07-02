# BikeSharing

This is an example of a Broadway application using RabbitMQ.

The idea is to simulate a bike sharing application that receives coordinates
of bikes in a city and saves those coordinates through a Broadway pipeline.

## Steps to reproduce

We first need to get a RabbitMQ instance running and with a valid queue created.

Using Docker, you can run the server by executing:

    docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management

This command will start the server and will block your terminal, so we need
to open another tab to created the queue. To do that we will enter in the container that is running:

    docker exec -it rabbitmq /bin/bash

And then, we create our queue:

    rabbitmqadmin declare queue name=bikes_queue durable=true

You can see that the queue "bikes_queue" was created by running `rabbitmqctl list_queues` in the
same window.

For this example you need the "postgis" extention for PostgreSQL.

### Running the app

After creating the queue, with RabbitMQ server running, you can execute the app in another window
with `iex -S mix`. It will be waiting for events.

To simulate events, first open a connection and then fire some messages:

```elixir
{:ok, connection} = AMQP.Connection.open
{:ok, channel} = AMQP.Channel.open(connection)
AMQP.Queue.declare(channel, "bikes_queue", durable: true)

Enum.each(1..5000, fn i ->
  AMQP.Basic.publish(channel, "", "bikes_queue", "message #{i}")
end)

AMQP.Connection.close(connection)
```

You can test with a sample set of data by running the script "priv/publish_sample_events.exs":

    mix run --no-halt priv/publish_sample_events.exs

#### Running with Docker Compose

If you don't want to install PostgreSQL or you don't want to run RabbitMQ by hand, you can try
to run this project using Docker compose.

First create the database:

    docker-compose run app mix setup

An then run the application:

    docker-compose up

It will take a while in the first time. You need to run `docker-compose build` everytime you
change a file in the project.


## Telemetry

Broadway currently exposes following Telemetry events:

  * `[:broadway, :topology, :init]` - Dispatched when the topology for
    a Broadway pipeline is initialized. The config key in the metadata
    contains the configuration options that were provided to
    `Broadway.start_link/2`.

    * Measurement: `%{time: System.monotonic_time}`
    * Metadata: `%{supervision: pid(), config: keyword()}`

  * `[:broadway, :processor, :start]` - Dispatched by a Broadway processor
    before the optional `c:prepare_messages/2`

    * Measurement: `%{time: System.monotonic_time}`
    * Metadata:

      ```
      %{
        topology_name: atom,
        name: atom,
        processor_key: atom,
        index: non_neg_integer,
        messages: [Broadway.Message.t]
      }
      ```

  * `[:broadway, :processor, :stop]` -  Dispatched by a Broadway processor
    after `c:prepare_messages/2` and after all `c:handle_message/3` callback
    has been invoked for all individual messages

    * Measurement: `%{time: System.monotonic_time, duration: native_time}`

    * Metadata:

      ```
      %{
        topology_name: atom,
        name: atom,
        processor_key: atom,
        index: non_neg_integer,
        successful_messages_to_ack: [Broadway.Message.t],
        successful_messages_to_forward: [Broadway.Message.t],
        failed_messages: [Broadway.Message.t]
      }
      ```

  * `[:broadway, :processor, :message, :start]` - Dispatched by a Broadway processor
    before your `c:handle_message/3` callback is invoked

    * Measurement: `%{time: System.monotonic_time}`

    * Metadata:

      ```
      %{
        processor_key: atom,
        topology_name: atom,
        name: atom,
        index: non_neg_integer,
        message: Broadway.Message.t
      }
      ```

  * `[:broadway, :processor, :message, :stop]` - Dispatched by a Broadway processor
    after your `c:handle_message/3` callback has returned

    * Measurement: `%{time: System.monotonic_time, duration: native_time}`

    * Metadata:

      ```
      %{
        processor_key: atom,
        topology_name: atom,
        name: atom,
        index: non_neg_integer,
        message: Broadway.Message.t,
      }
      ```

  * `[:broadway, :processor, :message, :exception]` - Dispatched by a Broadway processor
    if your `c:handle_message/3` callback encounters an exception

    * Measurement: `%{time: System.monotonic_time, duration: native_time}`

    * Metadata:

      ```
      %{
        processor_key: atom,
        topology_name: atom,
        name: atom,
        index: non_neg_integer,
        message: Broadway.Message.t,
        kind: kind,
        reason: reason,
        stacktrace: stacktrace
      }
      ```

  * `[:broadway, :consumer, :start]` - Dispatched by a Broadway consumer before your
    `c:handle_batch/4` callback is invoked

    * Measurement: `%{time: System.monotonic_time}`
    * Metadata:

      ```
      %{
        topology_name: atom,
        name: atom,
        index: non_neg_integer,
        messages: [Broadway.Message.t],
        batch_info: Broadway.BatchInfo.t
      }
      ```

  * `[:broadway, :consumer, :stop]` - Dispatched by a Broadway consumer after your
    `c:handle_batch/4` callback has returned

    * Measurement: `%{time: System.monotonic_time, duration: native_time}`

    * Metadata:

      ```
      %{
        topology_name: atom,
        name: atom,
        index: non_neg_integer,
        successful_messages: [Broadway.Message.t],
        failed_messages: [Broadway.Message.t],
        batch_info: Broadway.BatchInfo.t
      }
      ```

  * `[:broadway, :batcher, :start]` - Dispatched by a Broadway batcher before
    handling events

    * Measurement: `%{time: System.monotonic_time}`
    * Metadata:

      ```
      %{
        topology_name: atom,
        name: atom,
        batcher_key: atom,
        messages: [{Broadway.Message.t}]
      }
      ```

  * `[:broadway, :batcher, :stop]` - Dispatched by a Broadway batcher after
    handling events

    * Measurement: `%{time: System.monotonic_time, duration: native_time}`
    * Metadata: `%{topology_name: atom, name: atom, batcher_key: atom}
