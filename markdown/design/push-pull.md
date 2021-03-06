# push-pull

Normally a source element will push data to the downstream element using
the `gst_pad_push()` method. The downstream peer pad will receive the
buffer in the Chain function. In the push mode, the source element is
the driving force in the pipeline as it initiates data transport.

It is also possible for an element to pull data from an upstream
element. The downstream element does this by calling
`gst_pad_pull_range()` on one of its sinkpads. In this mode, the
downstream element is the driving force in the pipeline as it initiates
data transfer.

It is important that the elements are in the correct state to handle a
push() or a `pull_range()` from the peer element. For push() based
elements this means that all downstream elements should be in the
correct state and for `pull_range()` based elements this means the
upstream elements should be in the correct state.

Most sinkpads implement a chain function. This is the most common case.
sinkpads implementing a loop function will be the exception. Likewise
srcpads implementing a getrange function will be the exception.

## state changes

The GstBin sets the state of all the sink elements. These are the
elements without source pads.

Setting the state on an element will first activate all the srcpads and
then the sinkpads. For each of the sinkpads,
`gst_pad_check_pull_range()` is performed. If the sinkpad supports a
loopfunction and the peer pad returns TRUE from the GstPadCheckPullRange
function, then the peer pad is activated first as it must be in the
right state to handle a `_pull_range().` Note that the state change of
the element is not yet performed, just the activate function is called
on the source pad. This means that elements that implement a getrange
function must be prepared to get their activate function called before
their state change function.

Elements that have multiple sinkpads that require all of them to operate
in the same mode (push/pull) can use the `_check_pull_range()` on all
their pads and can then remove the loop functions if one of the pads
does not support pull based mode.
