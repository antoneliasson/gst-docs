# GstBus

The GstBus is an object responsible for delivering GstMessages in a
first-in first-out way from the streaming threads to the application.

Since the application typically only wants to deal with delivery of
these messages from one thread, the GstBus will marshall the messages
between different threads. This is important since the actual streaming
of media is done in another threads (streaming threads) than the
application. It is also important to not block the streaming threads
while the application deals with the message.

The GstBus provides support for GSource based notifications. This makes
it possible to handle the delivery in the glib mainloop. Different
GSources can be added to the same bin provided they listen to different
message types.

A message is posted on the bus with the `gst_bus_post()` method. With
the `gst_bus_peek()` and `_pop()` methods one can look at or retrieve a
previously posted message.

The bus can be polled with the `gst_bus_poll()` method. This methods
blocks up to the specified timeout value until one of the specified
messages types is posted on the bus. The application can then `_pop()`
the messages from the bus to handle them.

It is also possible to get messages from the bus without any thread
marshalling with the `gst_bus_set_sync_handler()` method. This makes
it possible to react to a message in the same thread that posted the
message on the bus. This should only be used if the application is able
to deal with messages from different threads.

If no messages are popped from the bus with either a GSource or
`gst_bus_pop()`, they remain on the bus.

When a pipeline or bin goes from READY into NULL state, it will set its
bus to flushing, ie. the bus will drop all existing and new messages on
the bus, This is necessary because bus messages hold references to the
bin/pipeline or its elements, so there are circular references that need
to be broken if one ever wants to be able to destroy a bin or pipeline
properly.
