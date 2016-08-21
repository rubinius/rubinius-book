# M

## Machine


## Memory

**These are rough notes that will be worked into documentation**
1. All allocation paths are GC-safe.

What that means is that when requesting a new object be created, the request
will be fulfilled *unless* the system (or process limits prevent it) *without*
GC running. In other words, there are two possible results of allocating an
object: 1) a new object, or 2) an exception because no more memory is
available to the process.

In either case, from the point the object is requested until that request
returns (or the return is bypassed by the exception unwind), the GC will not
run.

There is a trade-off here between running the GC at the instant that some
threshold is breached (eg the eden space is exhausted) and loosening some
requirements that must be maintained for a generational, moving garbage
collector (ie every object reference must be known to the GC at the time the
GC runs). Since we run GC on method entry and loop back branches, there is no
reasonable scenario in which deferring GC until allocation has completed will
result in unwanted object graph thresholds being breached pathologically (eg
an execution path where allocation can grow unbounded).

2. All objects are allocated from the various heaps *uninitialized* and a
protocol is established to call an initialization routine on the objects.

The initialization routine is `T::initialize(State* state, T* obj)`, where T
is
the type of object being allocated. The method is a static method of the class
of the object. This breaks with the protocol that Ruby uses where `new` is a
module method and `initialize` is an instance method. The primary reason for
choosing a static (ie C++ class) is to avoid an instance method operating on
an incompletely initialized object.

One purpose of this initialization protocol is to eliminate or reduce the
double initialization that we were doing (ie setting all fields to nil and
then initializing them to other default values). The main initialization
method shown above may be an empty body, in which case the compiler will elide
it anyway and there's no overhead to the protocol. In that case, another
initialization method should be called on the newly created object. Since the
allocation method is templated and if the initialization method is visible (ie
in the header file), the compiler should be able to elide remaining double
initialization in most contexts.
