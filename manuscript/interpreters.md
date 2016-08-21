# Interpreters - A Program at Play

A CallSite is an abstraction of the point of call & return for another
procedure. An InlineCache is an abstraction of a particular instance of an
executable at a CallSite.

In a 'statically typed' (or early bound) language, the executable at the call
site is known 'statically', via textual analysis of the program text. In a
'dynamically-typed' (or late bound) language, the particular executable is not
known until the program is executed. Consequently, in a late-bound language, a
call site must permit dynamic reconfiguration based on the program's
execution.

This dynamic reconfiguration requires prediction: when this kind of object is
seen, we predict we will see it again. The cost of finding the executable in
the first place can then be reduced by caching the previous result of looking
up the executable. When we see this kind of object, we can retrieve the
executable from the cache at less cost than looking it up.

If our predictions are correct, and the mechanism of creating and retrieving
items from the cache is actually less than searching for the item, we can
execute the program more efficiently. However, predictions can be wrong. If
they are wrong, the cache mechanism can become pure overhead, work done that
produces no value. In this case, the caching mechanism will make the program
*less* efficient rather than more efficient.

The number of cached items is typically limited because as the number grows,
the cost of retrieving items typically grows. Ideally, the first cached item
would always be the desired one. When that is not true, some (hopefully less
costly than lookup) search mechanism is required. Contiguous structures that
permit linear probing and are CPU cache-efficient usually perform very well.
If the number of entries grows, the algorithm needs to change from on O(n)
complexity to something closer to O(1), or the extra work of the caching
mechanism will be a cost increase instead of a cost reduction.

To account for our predictions being wrong, we need to be able to update,
replace, or even discard previous predictions. In same cases, we may need to
disable the caching mechanism completely. This latter case is true when the
types of objects seen at a call site vary widely.

This change also adds some new metrics for understanding how a program is
performing from the perspective of the call sites and inline caches. The next
mechanism to add is diagnostics to see the operation of individual caches
instead of the aggregate values that the metrics provide.

Another improvement that is needed is to cache `#method_missing`,
`#respond_to?` and `#send` call sites.
