# Stream Status

This document describes the design and use cases for the stream status
messages.

STREAM_STATUS messages are posted on the bus when the state of a
streaming thread changes. The purpose of this message is to allow the
application to interact with the streaming thread properties, such as
the thread priority or the threadpool to use.

We accommodate for the following requirements:

  - Application is informed when a streaming thread is about to be
    created. It should be possible for the application to suggest a
    custom GstTaskPool.

  - Application is informed when the status of a streaming thread is
    changed. This can be interesting for GUI application that want to
    visualize the status of the streaming threads
    (playing/paused/stopped)

  - Application is informed when a streaming thread is destroyed.

We allow for the following scenarios:

  - Elements require a specific (internal) streaming thread to operate
    or the application can create/specify a thread for the element.

  - Elements allow the application to configure a priority on the
    threads.

## Use cases

- boost the priority of the udp receiver streaming thread

```
        .--------.    .-------.    .------.    .-------.
        | udpsrc |    | depay |    | adec |    | asink |
        |       src->sink    src->sink   src->sink     |
        '--------'    '-------'    '------'    '-------'
```

- when going from READY to PAUSED state, udpsrc will require a
streaming thread for pushing data into the depayloader. It will
post a STREAM_STATUS message indicating its requirement for a
streaming thread.

- The application will usually react to the STREAM_STATUS
messages with a sync bus handler.

- The application can configure the GstTask with a custom
GstTaskPool to manage the streaming thread or it can ignore the
message which will make the element use its default GstTaskPool.

- The application can react to the ENTER/LEAVE stream status
message to configure the thread right before it is
started/stopped. This can be used to configure the thread
priority.

- Before the GstTask is changed state (start/pause/stop) a
STREAM_STATUS message is posted that can be used by the
application to keep track of the running streaming threads.

## Messages

The existing STREAM_STATUS message will be further defined and implemented in
(selected) elements. The following fields will be contained in the message:

  - **`type`**, GST_TYPE_STREAM_STATUS_TYPE:

      - a set of types to control the lifecycle of the thread:
        GST_STREAM_STATUS_TYPE_CREATE: a new streaming thread is going
        to be created. The application has the chance to configure a custom
        thread. GST_STREAM_STATUS_TYPE_ENTER: the streaming thread is
        about to enter its loop function for the first time.
        GST_STREAM_STATUS_TYPE_LEAVE: the streaming thread is about to
        leave its loop. GST_STREAM_STATUS_TYPE_DESTROY: a streaming
        thread is destroyed

      - A set of types to control the state of the threads:
        GST_STREAM_STATUS_TYPE_START: a streaming thread is started
        GST_STREAM_STATUS_TYPE_PAUSE: a streaming thread is paused
    GST_STREAM_STATUS_TYPE_STOP: a streaming thread is stopped

  - **`owner`**: GST_TYPE_ELEMENT: The owner element of the thread. The
    message source will contain the pad (or one of the pads) that will
    produce data by this thread. If this thread does not produce data on
    a pad, the message source will contain the owner as well. The idea
    is that the application should be able to see from the element/pad
    what function this thread has in the context of the application and
    configure the thread appropriatly.

  - **`object`**: G_TYPE, GstTask/GThread: A GstTask/GThread controlling
    this streaming thread.

  - **`flow-return`**: GstFlowReturn: A status code for why the thread state
    changed. when threads are created and started, this is usually
    GST_FLOW_OK but when they are stopping it contains the reason code
    why it stopped.

  - **`reason`**: G_TYPE_STRING: A string describing the reason why the
    thread started/stopped/paused. Can be NULL if no reason is given.

## Events

FIXME
