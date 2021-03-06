# Pad (de)activation

## Activation

When changing states, a bin will set the state on all of its children in
sink-to-source order. As elements undergo the READY→PAUSED transition,
their pads are activated so as to prepare for data flow. Some pads will
start tasks to drive the data flow.

An element activates its pads from sourcepads to sinkpads. This to make
sure that when the sinkpads are activated and ready to accept data, the
sourcepads are already active to pass the data downstream.

Pads can be activated in one of two modes, PUSH and PULL. PUSH pads are
the normal case, where the source pad in a link sends data to the sink
pad via `gst_pad_push()`. PULL pads instead have sink pads request data
from the source pads via `gst_pad_pull_range()`.

To activate a pad, the core will call `gst_pad_set_active()` with a
TRUE argument, indicating that the pad should be active. If the pad is
already active, be it in a PUSH or PULL mode, `gst_pad_set_active()`
will return without doing anything. Otherwise it will call the
activation function of the pad.

Because the core does not know in which mode to activate a pad (PUSH or
PULL), it delegates that choice to a method on the pad, activate(). The
activate() function of a pad should choose whether to operate in PUSH or
PULL mode. Once the choice is made, it should call ``activate_mode()`` with
the selected activation mode. The default activate() function will call
`activate_mode()` with ``#GST_PAD_MODE_PUSH``, as it is the default
mechanism for data flow. A sink pad that supports either mode of
operation might call `activate_mode(PULL)` if the SCHEDULING query
upstream contains the `#GST_PAD_MODE_PULL` scheduling mode, and
`activate_mode(PUSH)` otherwise.

Consider the case `fakesrc ! fakesink`, where fakesink is configured to
operate in PULL mode. State changes in the pipeline will start with
fakesink, which is the most downstream element. The core will call
`activate()` on fakesink’s sink pad. For fakesink to go into PULL mode, it
needs to implement a custom activate() function that will call
`activate_mode(PULL)` on its sink pad (because the default is to use PUSH
mode). activate_mode(PULL) is then responsible for starting the task
that pulls from fakesrc:src. Clearly, fakesrc needs to be notified that
fakesrc is about to pull on its src pad, even though the pipeline has
not yet changed fakesrc’s state. For this reason, GStreamer will first
call call `activate_mode(PULL)` on fakesink:sink’s peer before calling
`activate_mode(PULL)` on fakesink:sinks.

In short, upstream elements operating in PULL mode must be ready to
produce data in READY, after having `activate_mode(PULL)` called on their
source pad. Also, a call to `activate_mode(PULL)` needs to propagate
through the pipeline to every pad that a `gst_pad_pull()` will reach. In
the case `fakesrc ! identity ! fakesink`, calling `activate_mode(PULL)`
on identity’s source pad would need to activate its sink pad in pull
mode as well, which should propagate all the way to fakesrc.

If, on the other hand, `fakesrc ! fakesink` is operating in PUSH mode,
the activation sequence is different. First, activate() on fakesink:sink
calls `activate_mode(PUSH)` on fakesink:sink. Then fakesrc’s pads are
activated: sources first, then sinks (of which fakesrc has none).
fakesrc:src’s activation function is then called.

Note that it does not make sense to set an activation function on a
source pad. The peer of a source pad is downstream, meaning it should
have been activated first. If it was activated in PULL mode, the source
pad should have already had `activate_mode(PULL)` called on it, and thus
needs no further activation. Otherwise it should be in PUSH mode, which
is the choice of the default activation function.

So, in the PUSH case, the default activation function chooses PUSH mode,
which calls `activate_mode(PUSH),` which will then start a task on the
source pad and begin pushing. In this way PUSH scheduling is a bit
easier, because it follows the order of state changes in a pipeline.
fakesink is already in PAUSED with an active sink pad by the time
fakesrc starts pushing data.

## Deactivation

Pad deactivation occurs when its parent goes into the READY state or
when the pad is deactivated explicitly by the application or element.
`gst_pad_set_active()` is called with a FALSE argument, which then
calls `activate_mode(PUSH)` or `activate_mode(PULL)` with a FALSE
argument, depending on the current activation mode of the pad.

## Mode switching

Changing from push to pull modes needs a bit of thought. This is
actually possible and implemented but not yet documented here.
