A collection of discussions from @brixen via gitter.

there's definitely a role for blocking IO, but having a solid non-blocking option will be a huge advantage

I view blocking vs non-blocking IO and concurrency a lot like shared mutable state

sharing is fine, mutability is fine, shared mutable state is complicated

blocking IO is fine, concurrency is fine, blocking IO and concurrency is complicated

I'm going to add a new kind of Thread object that supports Actors that trampoline in their event loop, so that many Actors can be mapped onto a Thread pool, and code can transition from blocking IO to non-blocking IO

if an Actor's code makes a blocking call, the Thread won't be available in the pool and the execution stack is preserved

if the Actor's code does not make blocking calls, or completes the trampoline, it can be de-scheduled and the underlying Thread \(and stack\) can be reused

I'll try to get a PoC together soon, after finishing the transition to the new interpreter

the Actor mechanism combined with closed and isolated heaps is going to make simple concurrency much better supported

one change I'll need to make to Channel for the Actor thing I'm envisioning is that read\(\) cannot block the main event loop from returning or I won't be able to swap the Actor instances off the Thread

if we think of a Thread as T\(t, f\), where T is the managed object and t is the pthread, and f is the function that the pthread runs, and assume that when f exits, t is reclaimed by the OS and T is marked as a zombie \(ie it'll be GC'd as soon as there are no more references\)

think of an Actor as A\(e, m\) where A is the managed object, e is the main event look, and m is a method that represents the user's code

we want f to be able to call e1, e2, e3 for A1, A2, A3 in sequence, so when e1 is going to read from an empty mailbox \(ie channel\), we want e1 to return so that f can call e2

we think of e as having 3 possible return values: yield, completion, error

so, when e would read from a channel and "block", we instead want e to return the yield value to f

then f can check whether A\_n has a value in to be read before invoking e\_n for some n in the set of Actors indexed by i

so basically, we can use a couple conventions to make Actors potentially pretty simple and light weight, with a simple event loop that looks like a = mailbox.read; m\(a\)

ie read a message and dispatch it, where m would be a method that implements clauses to process various messages a

and with the isolated heaps I mentioned a while back, this works really nice

each Actor that uses an isolated heap has a heap pointer, which is an immix chunk, and the heap pointer is swapped in for the Thread-local allocator when the Actor runs, allocations happen in that chunk, isolated from any other Actor running on that Thread, but the Thread can gc itself independent of any other Thread

I'm going to work up a benchmark of Actors computing a mandelbrot set with one Actor per pixel and compare that with one Fiber per pixel on MRI

anyone want to place bets on who wins?

Chuck Remes @chuckremes Aug 26 2017 09:36

Makes sense. When I was trying to retrofit thread pools onto Celluloid I discovered that each Actor’s mailbox loop had a dedicated thread. There was no way to share. Right there it was clear that Celluloid couldn’t scale \(though I banged my head on that for 2 weeks before throwing in the towel\).

Brian Shirai @brixen Aug 26 2017 10:29

yeah, there's no way to be efficient when the Actor carries around a native thread

and there's no good reason to make that fixed

for a lot of reactive workloads, the event loop will very quickly read, dispatch, compute, loop

the other cool thing about this is that we don't actually interfere with the OS scheduler

we are merely multiplexing Actor computation onto a pthread

and there are these states for an Actor: waiting for a message, processing a message, or blocked processing a message

and for a Thread pool of N, Actors of M, we have N &lt; M, N = M, N &gt; M

unless N &lt; M, we don't even have to swap them

by dynamically growing and shrinking the pool, we can accommodate efficient parallelism on a particular machine

but that parallelism is uncoupled from the number of Actor instances that can be supported

Brian Shirai @brixen Sep 10 2017 23:31

hm, this is interesting, I thought this Actor idea would require some internal \(ie C++\) work, but so far, I'm writing it all in pretty basic Ruby

the non-blocking IO stuff will probably require some C++, and we need a ConditionVariable in core \(so the Actor Threads can suspend when there are no Actors to run\)

Brian Shirai @brixen Sep 16 2017 13:39

@chuckremes the more I play around with this Actor thing, the less useful anything with Fibers looks to me, just FYI

right now, the two things I need are a new Channel for Mailbox \(ie one that does not actually block on read when empty and instead will send a continuation msg\) and ConditionVariable in core so that the Actor::Thread can wait when there's no work to schedule

so far, I'm doing this all in pure Ruby, but I definitely think there's room to do some cool stuff with better CAS support, better native mutex and condvar support

Fibers are such an unnecessary complication in Ruby

Brian Shirai @brixen Sep 16 2017 14:31

@chuckremes oh yeah, one other thing, the Actor has a private heap pointer that the Thread allocator uses, which implies some Thread affinity for Actors with isolated or closed heaps that would require a heap copy if scheduled on another Thread

Chuck Remes @chuckremes Sep 16 2017 14:59

@brixen Not sure why you are comparing Actors to Fibers. Not to say this isn’t reasonable, but I’ve never heard anyone consider Actors to be similar at all to coroutines, fibers, green threads, or whatever else these execution contexts can be called. Usually Actors are created from fibers/coroutines; haven’t seen fibers or coroutines created from Actors.

Brian Shirai @brixen Sep 16 2017 15:00

Celluloid uses Fibers to implement Actors, no?

Chuck Remes @chuckremes Sep 16 2017 15:00

Celluloid uses a combination of Threads and Fibers to create an Actor. Trust me when I say it’s a total shit-show.

Brian Shirai @brixen Sep 16 2017 15:00

yeah, so what I'm saying is I have no use for Fibers in my Actor implementation

or anything useful I'd do in Ruby, period

I can implement coroutines with Actors very simply

Chuck Remes @chuckremes Sep 16 2017 15:02

Ok. I find them useful for creating the async-await concurrency abstraction. Need a way to capture program state. Fibers are a nice light-weight mechanism for doing so. I’m trying to figure out how to do the same with Actors as the underlying mechanism. Probably doable with some experimentation.

Brian Shirai @brixen Sep 16 2017 15:03

what I'm focusing on is 1. an Actor \(with its Mailbox\) decoupled from any stack \(ie Thread\); 2. a reasonable "client" interface to the Actor \(ie not what Elixir does\); 3. a non-blocking IO layer that effectively allows the Actor events to be multiplexed onto a reasonable sized Thread pool without blocking stack unwinding; and 4. doing all that without hacky stack switching

Chuck Remes @chuckremes Sep 16 2017 15:04

I like the plan!

Brian Shirai @brixen Sep 16 2017 15:04

sure, something like Fiber \(or much more simply, just a Thread\) is needed if the stack is being blocked, but that's the key thing I want to avoid

the migration path from undisciplined Ruby code that uses the legacy blocking IO is to allow it at the expense of efficiency \(an unnecessarily large Thread pool, or the possibility that nothing could progress because you run out of Threads\)

Chuck Remes @chuckremes Sep 16 2017 15:05

Thinking about it some more, an Actor sending/receiving via a mailbox is much like a Fiber sending/receiving via a Channel. Same ideas. Getting the program suspension/resumption to work is, as you said, the hard part.

Brian Shirai @brixen Sep 16 2017 15:05

yep, I think we're on the same page

Chuck Remes @chuckremes Sep 16 2017 15:06

Are you looking to have blocking IO that is async underneath with a blocking interface, or do you want separate IO mechanisms for async and sync to exist side by side?

Brian Shirai @brixen Sep 16 2017 15:07

separate

Chuck Remes @chuckremes Sep 16 2017 15:07

So something like SyncIO and AsyncIO parent classes? Programmer chooses their poison based on their use-case?

Brian Shirai @brixen Sep 16 2017 15:07

actually, all three is possible

Chuck Remes @chuckremes Sep 16 2017 15:08

Yeah, 1\) True sync, 2\) async, 3\) sync built on top of async.

Brian Shirai @brixen Sep 16 2017 15:08

the Actor mechanism could have a facility to use "blocking" IO \(blocking from the perspective of the code\), which would unwind and process the completion message, but it would require a bit more discipline

the 3rd one is maybe too much magic I think

Chuck Remes @chuckremes Sep 16 2017 15:09

So for “blocking” IO in an Actor, you could simulate it with an async mechanism that returns a Promise \(or Future\) and then blocks on it. The Actor scheduler would then swap out that execution context for another Actor that isn’t “blocked” on anything.

Brian Shirai @brixen Sep 16 2017 15:09

the idea is that "state" \(eg locals\) needs to be "heapified" but the stack frames can't contain non-managed state, because the frames are "going away" between the "blocking" call and the processing the "continuation msg"

all those "s signal a fair bit of complexity that could just be explicit

@chuckremes yeah, exactly that

Chuck Remes @chuckremes Sep 16 2017 15:10

heh

yes, state needs to be on the heap instead of the stack.

Brian Shirai @brixen Sep 16 2017 15:10

the IO mechanism would get the call with the Actor, and it would send a completion / continuation message to the Actor

instead of submerging that, it can be explicit, and I think the resulting code would be simpler

Chuck Remes @chuckremes Sep 16 2017 15:11

hmmm, that implies some magic within the actor though. When you make that IO call, you want that completion message to allow you to pick up on the “next line” of code inside the Actor.

Brian Shirai @brixen Sep 16 2017 15:12

for me, Actors are not some "wow" universal thing, instead they'd be used for "event-driven" code that easily encapsulates state and without the continuation spaghetti in JS

Chuck Remes @chuckremes Sep 16 2017 15:12

normally the boundaries for an Actor are at the “end” of the method. Run it to completion and then wait for another message to arrive in the mailbox. In this case, the “boundary” is when you make that blocking IO call too. It suspends there.

Brian Shirai @brixen Sep 16 2017 15:12

@chuckremes yep, but that's possible because the whole scope chain, including the current instruction pointer could be reified on the heap

there just cannot be any non-managed code in the stack

in Rubinius, all the locals are in a shadow stack

Chuck Remes @chuckremes Sep 16 2017 15:13

I agree that your plans are doable. Kudos if you are able to pull most of it off in Ruby without resorting to C.

Brian Shirai @brixen Sep 16 2017 15:14

if the IP \(current location in the bytecode\) were also on that stack, you can "rehydrate" a machine stack without any hacky code

we have this in the old JIT with the uncommon\_interpreter

it would jump from machine code back into a certain location in an interpreter function instead of at IP = 0

Chuck Remes @chuckremes Sep 16 2017 15:14

ah… I think I grok that.

Brian Shirai @brixen Sep 16 2017 15:15

what this is really is the value of being sure that 100% of the semantics are represented in the instruction set, instead of submerged in opaque "primitives"

and that's why I've been working so much on the interpreter \(and eventually the JIT\)

Brian Shirai @brixen Sep 30 2017 09:38

@chuckremes also considering adding ByteBuffer and going to immutable String so that Strings can be values in Actor messages, avoiding copy even between isolated heaps

Chuck Remes @chuckremes Sep 30 2017 11:50

RE: immutable strings. Yes, please.

RE: ByteBuffer. I’m not sure I know what you mean here, but if it’s a first-class object for handling byte streams \(no encodings or other overhead\) then, Yes, please.

Brian Shirai @brixen Sep 30 2017 15:04

@chuckremes what I'm thinking for ByteBuffer is an updatable \(indexed and append\) of bytes that can carry an encoding attribute, but does not use it

this allows building up a ByteBuffer from external bytes that are said to have an encoding and then converting that to a String, possibly with transcoding, that has a particular encoding

as a transition step, I'm considering essentially doing this Rubinius::ByteBuffer = ::String; class String ... end

and creaing a new String that has no mutation

this would allow people who are still mutating their Strings to do the reverse temporarily \(ie String = Rubinius::ByteBuffer\)

now that the new instruction set is in, I need to fix keywords so that we use an explicit mechanism for passing them and call sites with fixed arity have zero overhead for using keywords

for sites that use \*\*kw or for methods that accept \*\*kw, we can also improve performance significantly

Brian Shirai @brixen Oct 02 2017 23:51

@chuckremes beyond immutable String and with new IO APIs, a more impactful change would be to push any encoding to only IO boundaries and have only utf-8 encoding in the runtime

this could enable significant String performance optimizations

the initial argument that not all encodings could be transcoded to utf-8 is no longer true with recent unicode versions, I think

Chuck Remes @chuckremes Oct 03 2017 07:09

haha, great. Baby steps are still steps.

You’ve long talked about IO using utf-8 internally and only transcoding once at the boundaries. Makes a lot of sense to me.

However it kind of implies that there needs to be an “internal only” IO API that the class itself can use to talk to itself. If we start composing internal functions using boundary functions then we’ll end up \(potentially\) transcoding multiple times. The public external facing API should be the only one that transcodes.

I’ve started pondering how to do a “functional core & imperative shell” for IO. Needs more research but I’ll share some thoughts here later today or this week.

Chuck Remes @chuckremes Nov 03 2017 13:36

@brixen Wanted to bounce a couple of things off of you. I’ve been thinking about an IO rewrite and the goal of keeping Encodings at the boundaries.

Brian Shirai @brixen Nov 03 2017 13:37

ok

Chuck Remes @chuckremes Nov 03 2017 13:37

I figured the core classes would only deal with ASCII\_8BIT so that the read/write methods are as fast an unencumbered as possible.

Brian Shirai @brixen Nov 03 2017 13:37

I'm debating two approaches: 1. utf-8 only; 2. the internal encoding is set at boot time

Chuck Remes @chuckremes Nov 03 2017 13:37

When someone wanted to work with a non-binary encoding, they would essentially wrap their IO calls with calls to Encodings.convert\(buffer: buf, target\_encoding: Encodings::WHATEVER\)

The call to Encodings.convert would be a function. Given same input, always get same output \(or an exception\).

Brian Shirai @brixen Nov 03 2017 13:38

I'm leaning toward 1 because utf-8 is suitable for anything these days \(afaik\), but 2 would permit the same optimizations as 1 an allow a user to pick something else if they needed to

for Ruby, I wasn't going to change the way IO is done, only that you cannot change \(or specify\) and internal encoding

Chuck Remes @chuckremes Nov 03 2017 13:39

Thinking...

Brian Shirai @brixen Nov 03 2017 13:40

I'm open to other APIs in Rubinius, but for exposing to Ruby, I was trying to standardize on the most common pattern \(ie 1 internal encoding, and specify the transcoding to occur when reading or writing by using the external encoding option if it is different than the internal one\)

there are plenty of languages that just support utf-8, and standardizing on that as the only internal encoding is pretty defensible

Chuck Remes @chuckremes Nov 03 2017 13:40

If you go with 2, then aren’t you always paying the toll to transcode even when the target encoding is not multi-byte?

I’d just like to get encodings out of the hot path if someone is writing \(for example\) network code that is blasting binary all over the place.

Brian Shirai @brixen Nov 03 2017 13:42

so, in that case, I'd use the ByteBuffer class

it is just bytes

Chuck Remes @chuckremes Nov 03 2017 13:42

is that a new yet to be introduced class? So, IO would no longer use String?

What I should do is sit down and write up the README/documentation for my ideas complete with API. Then we would have something concrete to discuss.

So, never mind for now.

Brian Shirai @brixen Nov 03 2017 13:47

ok

Chuck Remes @chuckremes Nov 03 2017 13:47

Question though… does anything new I come up with need to be 100% backward compatible with existing Ruby IO API?

Brian Shirai @brixen Nov 03 2017 13:49

I don't think so

one sec, I'll give you a more complete example

Chuck Remes @chuckremes Nov 03 2017 13:49

pls

Brian Shirai @brixen Nov 03 2017 13:56

sorry, had to finish something...

let's be sure we separate two things: 1. Rubinius APIs, classes, instructions, etc.; and 2. Ruby compatibility

Chuck Remes @chuckremes Nov 03 2017 13:58

lsitening...

Brian Shirai @brixen Nov 03 2017 13:58

for 2. I want these things in the near future: 1. immutable String and mutable ByteBuffer; 2. a single, non-mutable internal encoding; 3. better IO that allows for async IO to not block my Actor execution pool

for 1. I want the best ideas we can come up with

one thing I know about IO is that I don't want sync or async to be subordinate to the other

Chuck Remes @chuckremes Nov 03 2017 13:58

so 2 will use the classes from 1?

Brian Shirai @brixen Nov 03 2017 13:59

I know you can build async on top of sync \(use threads\) or sync on top of async \(introduce blocking\), but I don't want either of those

I want paralell IO mechanisms where one is optimized for sync and one optimized for async

Chuck Remes @chuckremes Nov 03 2017 13:59

Yes, agreed. Though some things are inherently blocking such as file read/write, file open. They can be nonblocking for pipes & sockets but NOT for files.

Brian Shirai @brixen Nov 03 2017 13:59

and I want it visible in the program that one or the other is being used, rather than "magic" \(eg em-synchrony\) happening under the covers

Chuck Remes @chuckremes Nov 03 2017 14:00

SyncIO and AsyncIO classes as peers.

Brian Shirai @brixen Nov 03 2017 14:00

yeah

Chuck Remes @chuckremes Nov 03 2017 14:00

we are definitely on the same page then \(so far!\)

Brian Shirai @brixen Nov 03 2017 14:00

ok, cool

sounds like you mostly want to work on the SyncIO, AsyncIO \(ie Rubinius facilities\), is that right? or are you also concerned about the Ruby compatibility side?

Chuck Remes @chuckremes Nov 03 2017 14:01

Note: any AsyncIO class would need a small thread pool to handle blocking activities like opening a file, stat’ing a file, read/write a file.

Brian Shirai @brixen Nov 03 2017 14:01

yeah, so that small thread pool thing is also something we need for Actor execution pool, and I'm sure other things, so I want to see where and how that needs to be a first class construct

Chuck Remes @chuckremes Nov 03 2017 14:01

I want to work on SyncIO/AsyncIO and maybe provide a shim layer so that Ruby IO uses SyncIO behind the scenes.

Brian Shirai @brixen Nov 03 2017 14:01

ok, that sounds good

the other thing, then, is that most of these facilities are going to fundamentally be new instructions

that's consistent with the shift away from primitives so we can bring compiler technology to bear for optimizing, and so we can clearly target something like WebAssembly

Chuck Remes @chuckremes Nov 03 2017 14:03

Where things get hairy with providing compatibility shims is when someone wants to open a socket O\_NONBLOCK and then use read\_nonblock/write\_nonblock. The SyncIO class really shouldn’t provide that capability \(i.e. nonblocking stuff\).

Brian Shirai @brixen Nov 03 2017 14:03

I'm ok breaking compatibility with stuff like that because there's no real benefit to supporting it

"I want an apple, but please make it taste like a banana"

no, go eat a banana

Chuck Remes @chuckremes Nov 03 2017 14:04

So, your reference to new instructions ^^ is where a concrete example would be helpful.

Brian Shirai @brixen Nov 03 2017 14:04

well, for now, work on the semantics and classes, the instruction stuff will be premature

Chuck Remes @chuckremes Nov 03 2017 14:04

I had planned on writing all of this in Ruby + FFI. Instructions are, by definition, in the platform language like C++ or WebAssembly, right?

Brian Shirai @brixen Nov 03 2017 14:05

but it will be obvious where we need to enhance based on two things: 1. where we have to reach for primitives; 2. where optimization is really hard

so, FFI is a great start because I'm going to be pulling FFI up into the instructions

but I'm also investigating things like a create\_thread instruction

so, keep the instruction stuff in the back of your mind, but go for the semantics you want

Chuck Remes @chuckremes Nov 03 2017 14:06

ok

I’m going to take a few days to write up some docs for the SyncIO and AsyncIO classes. I’ll share it here for comment and discussion.

Brian Shirai @brixen Nov 03 2017 14:06

in fact, I'll take a pass through your recent work on IO and see where FFI and other instructions could be immediately leveraged

that sounds great, super exciting

the thing I'm focused on is really accelerating new system development, and not wasting a single cycle on trying to support legacy Ruby

Chuck Remes @chuckremes Nov 03 2017 14:08

Just need someone with vision to help bankroll this. As soon as I’m gainfully employed this will move to the back burner.

Brian Shirai @brixen Nov 03 2017 14:08

I don't want to willy-nilly break compatibility, but compatibility is a cost that must have a higher reward than its cost

@chuckremes oh, this may seem obvious, but ignore the completely silly relegation of Socket to stdlib

that's a relic of 1993 that we definitely don't want to drag into 2017

IO should be in the system 100%

and that should completely extricate you from the bizarre Socket, IO, File hierarchy that exists in Ruby

also, I've been revisiting ZeroMQ and nanomsg and ultimately we need to support the concepts in nanomsg

that is fundamental infrastructure when you have small communicating processes

Chuck Remes @chuckremes Nov 03 2017 14:14

Heh, don’t worry about me slavishly following Ruby’s broken IO inheritance tree. Never. In. A. Million. Years.

Brian Shirai @brixen Nov 03 2017 14:14

haha, awesome

Chuck Remes @chuckremes Nov 03 2017 14:14

We need IO::File, IO::Pipe, IO::TTY, IO::Directory, IO::Socket, IO::WebSocket, IO::Zeromq \(or whatever\), etc.

Brian Shirai @brixen Nov 03 2017 14:14

yep, seems good

the point from ZeroMQ/nanomsg is that a service A needs to be able to talk to a \(possibly set of\) service B and should be able to use an IO mechanism that can shift from in-process to IPC to inter-node without the application code caring

there should be an very clear and visible distinction between method / function call and communication, but the communication could be inter-Actor \(ie in-process\) or it could be inter-process \(ie IPC on that compute node\) or it could be inter-node \(ie TCP, HTTP, etc\)

Chuck Remes @chuckremes Nov 03 2017 14:17

If I’m to achieve my vision for distributed computing, that’s a must have.

Brian Shirai @brixen Nov 03 2017 14:17

very good, carry on sir

Chuck Remes @chuckremes Nov 03 2017 14:17

!!

And a blockchain-based registry for registering and finding distributed actors

Brian Shirai @brixen Nov 03 2017 14:18

that, indeed, is a very cool idea

Chuck Remes @chuckremes Nov 03 2017 14:18

\(pretty much an ideal use of blockahin\)

Brian Shirai @brixen Nov 03 2017 14:18

I'm starting with erasing packaging nonsense by leveraging IPFS \(or conceptually similar\), but the idea of moving that up to services is compelling

Chuck Remes @chuckremes Nov 03 2017 14:18

Oh, we’ll probably also need IO::IPFS, yes?

haha, messages crossed

Brian Shirai @brixen Nov 03 2017 14:18

haha, sure

convergence

Chuck Remes @chuckremes Nov 03 2017 14:19

we’re in a similar headspace

Brian Shirai @brixen Nov 03 2017 14:19

indeed

Chuck Remes @chuckremes Nov 03 2017 14:21

Then I need to make this new IO completely thread-safe. I’m thinking a lock around every method. Granular!

j/k

ok, enough messing around. Time to write some docs.

Chuck Remes @chuckremes Nov 03 2017 14:36

@brixen Just an aside… answer whenever… do you prefer IO operations to raise Exceptions or to return an error code/object? That is, should calling IO::Socket\#seek return NotImplementedError or an IOError object with a code, reason, backtrace, etc?

Brian Shirai @brixen Nov 03 2017 15:32

@chuckremes my bias is toward exceptions, and the reason is this: when you pass error objects, you couple the location the error is raised through all intervening code to the location the error is handled, and having done lots of C code, I found this to be a big problem

I totally get the nanomsg author's criticism of C++ and undefined behavior around exceptions and constructor/destructor code, but I'm not convinced for a decent high-level language that doesn't have to inter-operate with C

now, since these IO method calls should probably be \(near\) leaf method calls, providing one that raises and one that returns an error object isn't unreasonable

let the user decide which mechanism they want

but we'd need to come up with a pretty decent way to implement that

one thing that comes to mind immediately is a "policy" class and depending on which error policy you want, it may raise or return an object

the way the implementation code would handle that is something like this

class SyncIO

def read\(bytes:, sink:\)

```
\# ...

return client.error\_policy.new\(:read, blah\)
```

end

end

obviously here, client looks kinda magical because there could be more than one "client" in a process

Brian Shirai @brixen Nov 03 2017 15:37

the constructor \(ie error\_policy.new\) would construct the error object and either return it, or raise it

anyway, let me know what you think

Chuck Remes @chuckremes Nov 03 2017 15:38

Hmmm, not a terrible idea.

Brian Shirai @brixen Nov 03 2017 15:38

@chuckremes if we assigned a policy to every "IO initiation" method, this becomes pretty easy

it's just state on that IO

eg IO.new channel: xyz, policy: :error\_object

I'm just spit-balling, but I think this would work well

Chuck Remes @chuckremes Nov 03 2017 15:39

Right. SyncIO.error\_policy=\(policy\_struct\)with some sensible default. New instantiations would just use the currently set SyncIO.error\_policy during their construction.

Brian Shirai @brixen Nov 03 2017 15:40

for default, sure, and by-instance overrides

Chuck Remes @chuckremes Nov 03 2017 15:40

Right. I can live with that. I’m doing something similar with SyncIO.mode and SyncIO.flags since there is usually a sensible default.

Brian Shirai @brixen Nov 03 2017 15:40

one thing I'd suggest is looking at this though a "capabilities model" lense

I think there's a nice application here

Chuck Remes @chuckremes Nov 03 2017 15:41

Not familiar with that term. I’ll google for it.

Brian Shirai @brixen Nov 03 2017 15:41

if we think more explicitly in terms of a capabilities model, the error policy fits in there

one sec, I'll link you to a great resource

very 1993 website though lol

Chuck Remes @chuckremes Nov 03 2017 15:41

This? [https://en.wikipedia.org/wiki/Object-capability\_model](https://en.wikipedia.org/wiki/Object-capability_model)

Brian Shirai @brixen Nov 03 2017 15:42

[http://www.erights.org](http://www.erights.org)

yeah, that wikipedia link too

Mark Miller has done some really cool work on it

Chuck Remes @chuckremes Nov 03 2017 15:42

I’ll look to these for inspiration.

Brian Shirai @brixen Nov 03 2017 15:42

the Mirror stuff and the capability model stuff are super interesting

and I want to bring a capability model in

ie, system shouldn't be accessible at any arbitrary point in code

instead, you should need access to a capability policy object that would or would not have a system method

this paper is a must-read [http://www.erights.org/talks/thesis/index.html](http://www.erights.org/talks/thesis/index.html)

but don't let this stuff derail you, there's lots of opportunity to enhance

I'm working on a capability-like model for the instruction set, so that eg you cannot bind a method to a class if the method uses mutating instructions and the class says it's not mutable

this will allow us to make String immutable, still allow arbitrary compilation of code, but when you define a String method, you can't use any bytecode that can modify it

this doesn't require changing any Ruby semantics other than that defining a method may raise an ImmutableClassError or something

Chuck Remes @chuckremes Nov 03 2017 15:52

I’ll read that thesis sometime in the next week. I want to get my rough thoughts down on the class API and semantics. Then we can iterate. Thanks for the pointer.

Chuck Remes @chuckremes Nov 03 2017 16:01

200 pages? I sometimes forget that dissertations are judged by weight. /snark

Chuck Remes @chuckremes Nov 03 2017 16:16

apropos of nothing, I’m thinking that instead of select we offer an epoll-like API. With that API we can mimic a select or kqueue API. Just thinking out loud here.

Chuck Remes @chuckremes Nov 03 2017 16:59

Continuing that thought, looks like the kqueue API is much more functional than epoll. And epoll is more easily mimicked via kqueue than vice versa. Anyway, this API is only necessary for dealing with external file descriptors. The AsyncIO class should be capable of hiding all of that event-driven complexity so select or epoll or whatever should be unnecessary in 99% of cases.

Chuck Remes @chuckremes Nov 10 2017 17:20

@brixen I know from earlier conversations you really want Sync versions of IO operations. Is that specific to File IO or does that extend to Socket IO too? That is, do you want Sync/Async File IO and Sync/Async Socket IO, or just Sync/Async File IO with Async-only Socket IO?

I think I know the answer \(the first one\) but I want to double check.

Brian Shirai @brixen Nov 10 2017 17:33

@chuckremes I think Sync vs Async is an application level decision, so I'd expect the infrastructure \(File, Socket, etc\) to provide both on equal footing

it's the application that knows the tradeoffs for selecting sync vs async, to put it another way

does that answer your question @chuckremes ?

Chuck Remes @chuckremes Nov 10 2017 18:18

yep

I’ve been pondering how to make the async api look completely synchronous via fibers. That is actually pretty easy to accomplish.

What’s hard is making sure that other fibers in the same thread cooperate. That is quite impossible unless the runtime enforces some kind of yield. Imagine a thread that is running hundreds of fibers handling as many incoming socket connections. One connection advances through its protocol state machine and goes into an infinite loop. This blocks all other fibers on that thread from ever advancing. Oops.

I know that programmers can sometimes make mistakes but if we adhere to my Ruby philosophy then the language runtime should save you from yourself. Maybe that’s misguided. I don’t know.

@brixen Your thoughts on ^^ would be welcome. I’ll check back later.

Brian Shirai @brixen Nov 10 2017 19:26

this is such a good talk, I re-watch it periodically and every time I do, I think I should have re-watched it sooner [https://www.youtube.com/watch?v=O3tVctB\_VSU](https://www.youtube.com/watch?v=O3tVctB_VSU)

@chuckremes ^^^ you will really enjoy this

@chuckremes so RE your question, for Actors on Rubinius, I'm not thinking in terms of Fibers at all

in fact, I don't want Fibers in the mix

I started some code what I think Actors should be like on Rubinius, and was a bit surprised to see that it doesn't need much support, other than an IO subsystem that knows about the Actors so that an IO call won't block the Actor message processing loop

I should probably finish coding that up so you can see it before talking more about it because that will be easiest way to check our understanding

one other thing I should mention: I'm not interesting in supporting some arbitrary "anything goes" behavior

Brian Shirai @brixen Nov 10 2017 19:31

I expect the message processing loop to run like an event loop, ie short, complete computations that essentially trampoline and can permit the Actor message loop to be suspended in the common case

in the uncommon case that a blocking call occurs, that Thread is basically parked, and another Thread is added to the pool if demand is high and thread resources are available, or the pool may exhibit pathological \(exhausted resources\) behavior

I'm not trying to prevent the latter. Let it fail and let it get nixed and restarted

Chuck Remes @chuckremes Nov 10 2017 21:00

Okay, interested to see some code \(pseudo-code is fine\). If we are to have non-blocking IO but code still appears to run synchronously, I’ll be interested to see how we achieve that without fibers. But then, I’m thinking that each real thread should be able to run multiple callstacks \(coroutines, fibers, whatever\) for concurrency \(though not parallelism\). My use-case is a thread that needs to handle 500 TCP sockets. How do we do that with the Actor on a single thread without fibers.

Thanks for the link to the video. I’ll watch it this weekend.

Brian Shirai @brixen Nov 10 2017 21:10

this talk is also so good [https://www.youtube.com/watch?v=NdSD07U5uBs](https://www.youtube.com/watch?v=NdSD07U5uBs)

@chuckremes so, in the case of the 500 TCP connections, my view is there should be 500 Actors that service a connection each, and the overhead for each connection is this: size of Actor + size of TCP connection + a little bit of state for the message loop and the binding to a Thread pool

the Actors themselves do not bind to a stack \(eg Thread\), but are multiplexed onto the Thread pool

500 Actors servicing TCP connections is a pretty small set of resources, and will achieve maximum parallelism without wasting "parallel potential" resources \(ie threads\)

Chuck Remes @chuckremes Nov 10 2017 21:14

So the Actor maintains the “state” much like a fiber or coroutine. Is this just an issue of semantics?

Brian Shirai @brixen Nov 10 2017 21:15

the IO is important here, because read or write operations on those TCP connections can't block \(or you'd waste a ton of Threads\), so the operation needs to turn itself into a continuation delivered to the message processing loop, and allow the loop to unwind to a place where the actor can be swapped off the Thread

the Actor does not maintain any "stack" state

Chuck Remes @chuckremes Nov 10 2017 21:15

ok. what would a backtrace look like in the event of a failure?

Brian Shirai @brixen Nov 10 2017 21:16

but it may maintain "internal" state \(ie what would determine the Actor's behavior processing the next message, whether it sends a message, changes internal state, or creates a new Actor\)

what sort of failure are you thinking of?

Chuck Remes @chuckremes Nov 10 2017 21:16

this is Ruby, so let’s say we run into a NoMethodError somewhere along the way.

Brian Shirai @brixen Nov 10 2017 21:17

ok, a NoMethodError would unwind the Actor's stack, and result in an exception state for that Actor, which would cause it to get descheduled on the Thread it's running on

the semantics can be defined, obviously, but what I was going to do is if the Actor code unwound to the underlying message processing loop, the Actor would "die", notifying any monitors

other behavior is possible, but the idea is that the "top" of the stack is the Actor's own loop, not the underlying Thread it is running on

Chuck Remes @chuckremes Nov 10 2017 21:18

all right, I think I get it. Each Actor has its own mailbox loop going, so in effect each Actor has its own event loop \(mailbox =&gt; event-loop\). It’s processing messages off that mailbox, so if a message leads to an Exception \(like NomethodError\) then the stack trace would lead back to the “pop” operation off the mailbox.

Brian Shirai @brixen Nov 10 2017 21:19

yes

that's exactly the model I'm going for

Chuck Remes @chuckremes Nov 10 2017 21:19

That’s reasonable.

So loop do … end structures within an Actor will be contra-indicated.

Brian Shirai @brixen Nov 10 2017 21:19

the Actor itself defines which "messages" it will understand, or a catchall that will have to figure out what to do, but it doesn't own the loop

the loop itself is implemented by the underlying Actor mechanism, and when the loop runs, it can "suspend", causing the Thread pool to schedule a new actor on a particular Thread

Chuck Remes @chuckremes Nov 10 2017 21:21

This implies that not all objects in the system are Actors. Some objects may have blocking properties. Those objects can send messages to Actors \(and maybe block on a response\), but the contra isn’t possible wherein an Actor sends a message to a non-Actor and blocks on a response. All Actor interactions are async.

just thinking out loud.

Brian Shirai @brixen Nov 10 2017 21:21

the IO needs to be able to say, "ah, this is Actor instance xyz, I'm going to block, so I'll send a continuation message to xyz's message box and return" and that return should propagate back up the stack until that message processing loop invocation has returned to the message loop mechanism, which notifies the Thread scheduler

so this is a really impure system

I'm relying on two things to get decent behavior, 1. convention, and 2. a better underlying mechanism being available

Chuck Remes @chuckremes Nov 10 2017 21:22

\#1 is important. It’s actually key.

Brian Shirai @brixen Nov 10 2017 21:23

the blocking calls would in fact tie up that Thread, and may be needed for transitioning to better code, but it's 100% possible to program "correctly" so that blocking calls are not used

Chuck Remes @chuckremes Nov 10 2017 21:23

If you can whip up some pseudo-code in your copious free time, I can use that as a baseline for my new SyncIO/AsyncIO docs that I’m working on. I’ll make sure it fits within your construct.

Brian Shirai @brixen Nov 10 2017 21:23

ok, I did start on some code, so let me see if I can get that functional this weekend

Chuck Remes @chuckremes Nov 10 2017 21:24

Honestly I was hoping for a system that might work on other Ruby runtimes “out of the box” without special support like you are describing.

Brian Shirai @brixen Nov 10 2017 21:24

it uses what I think are some nice conventions to make writing the actual Actor-relevant behavior easier

so, this should work in a basic way, except that the IO mechanism wouldn't be there \(but could be faked a bit\)

but it's 100% outside of my interest to make it cross-Ruby

Chuck Remes @chuckremes Nov 10 2017 21:25

I was looking to do a Thread each with its own Fiber scheduler. When a “blocking” IO call was made, the IO object would message an IO thread to handle it and then signal the current thread to suspend this Fiber until the result came in. Upon suspension, the Thread could schedule other fibers to run.

Brian Shirai @brixen Nov 10 2017 21:25

that said, when I started working on it, I was surprised that no really Rubinius-specific mechanism was required to make it work the way I thought it should

\(except the IO support\)

so, the biggest consideration of my scheme is avoiding the binding of an Actor to a stack abstraction \(Thread or Fiber\)

Chuck Remes @chuckremes Nov 10 2017 21:26

I’m starting to understand that.

Brian Shirai @brixen Nov 10 2017 21:27

it means that you have to write "event-oriented" code, but what seems pretty obvious to me is that it's that kind of code that's going to result in less complexity and better composability

which takes me back to 1. above \(ie convention\)

Chuck Remes @chuckremes Nov 10 2017 21:28

I’ll wait for some pseudo-code \(or actual code\). I’m not super-keen on the idea of writing evented code. I want language support to make the evented code look like imperative \(or procedural or sequential or …\) code.

Brian Shirai @brixen Nov 10 2017 21:28

also, something that I'm still thinking about is the ability to "defer" \(wrap up some computation and send it as a message, like a continuation\)

but maybe that's just the message sending mechanism \(ie sending a message to yourself\)

Chuck Remes @chuckremes Nov 10 2017 21:29

Sure, it’s allowed for Actors to message themselves.

Brian Shirai @brixen Nov 10 2017 21:29

the "evented" code is just processing a set of messages in a very fine-grained manner

the sort of "threading" that may seem natural becomes a distince stream of messages

if that stream needs an ordering, I think that can be provided, but I'm uncertain it's often necessary

this sort af mechanism models a state machine quite explicitly

and, as it turns out, is exactly like a set of microservices

these "messaging" architectures turn out to be very similar

and quite in contrast to the spaghetti code complexity of "typical" more monolithic code

Chuck Remes @chuckremes Nov 10 2017 21:32

Yeah, I’d like to replace state machines with coroutines/fibers. Easier to reason about at a glance.

This topic brings to mind the best message I ever wrote to a mailing list. Here: [https://groups.google.com/forum/\#!msg/celluloid-ruby/htqsuhCLXRw/1NI6C-07BQAJ](https://groups.google.com/forum/#!msg/celluloid-ruby/htqsuhCLXRw/1NI6C-07BQAJ)

Brian Shirai @brixen Nov 10 2017 21:32

one thing that might help is if you could describe an app for me, and I'll implement it

I was going to use a parallel computation of a Mandelbrot set \(ie one Actor per pixel\) to demonstrate this, but you may have a more realistic app in mind

Chuck Remes @chuckremes Nov 10 2017 21:34

I do think I’m a few degrees off from understanding your points. If you could implement a simple TCP server that listens on a socket and does some computation on an incoming message \(computation of arbitrary complexity and length\), then maybe I’d understand better.

Sorry, gotta run for tonight. I’ll be online this weekend.

Brian Shirai @brixen Nov 10 2017 21:36

cool, I'll rewatch this Hewitt video

I think where I'm coming from on this is that the "long running computation" can be broken up internally by the Actor in a way that it does not need to abstract to the "stack", and can allow the Actor to be multiplexed onto Threads in a way that reasonable parallelism is possible

any "long running computation" either 1. includes blocking calls \(ie occupies the stack but makes no progress\); or 2. makes looping "progress" \(ignoring combinations of 1 and 2 because i'll just decompose them and recurs\)

handling 1. requires some underlying \(eg IO\) mechanism to cooperate and not block the stack

and 2 can be handled with a little bit of \(some kind of\) sugar

loops \(ie iteration\) being expressible as recursion motivates loops being expressible as a sequence of messages

the only other sort of "long running computation" would be a linear stack deepening \(ie really really deep\) and that's going to abort without TCO anyway

so the focus really is back to 1 and 2

Brian Shirai @brixen Nov 10 2017 21:42

for 1, we need IO to be able to say "hey, Actor xyz called me, I'm not going to block, but instead send a completion message to xyz", and for 2, we need a nice way to break that looping into messages that let the stack unwind and the Actor be de/re-scheduled

Brian Shirai @brixen Nov 11 2017 15:26

this whole talk is great, but this section especially about the power of language [https://youtu.be/YyIQKBzIuBY?t=2986](https://youtu.be/YyIQKBzIuBY?t=2986)

Chuck Remes @chuckremes Nov 12 2017 08:07

@brixen I’ll be online a bit later today. I’ve been pondering how an Actor could behave the way you describe above \(with stack unwinding every time it hits IO\) but I’m having a failure of imagination. I can’t see how that stack unwinding would work and allow the Actor to pick up where it left off without every Actor needing to implement its own state machine.

If it yielded in a Fiber, then the callstack would be preserved and the Actor could pick up where it left off without any additional work on the part of the programmer.

So, if you could pseudo-code an example showing how this would work I would be grateful. Catch ya later.

Brian Shirai @brixen Nov 12 2017 11:05

@chuckremes so, the biggest thing to keep in mind is that I'm suggesting a different discipline for writing these Actors, and I'm explicitly calling out the "laziness" \(in a sense\) of assuming that the stack abstraction should be arbitrarily available

it's a resource, and it should be use when it is "essential" \(ie not when it can't be done otherwise, since this is Turing Land™, but when it makes the solution "better"\), but not assumed to be 1. without cost; 2. invisible

so, keep in mind that I'm both learning about the intricacies of Actors \(as Hewitt really defines them, not as they've been "interpreted"\), as well as trying to bring some deconstruction to the assumptions we use regularly

that's why I'm suggesting a hybrid where an Actor does block, consume that stack resource \(ie Thread\), and the system tries to adapt by pulling in more Thread resources or by applying a sort of "back pressure" to the request for computation resources

as a very simple example, assume this sequence of calls A -&gt; B -&gt; C -&gt; blocking\_IO ... at C, it's highly unlikely we want to block, and if the IO call didn't block, C would continue, B would continue, A would continue \(let's call this point 1. so we can discuss later\)

Brian Shirai @brixen Nov 12 2017 11:10

point 2 would be that in either A or B or C \(or all of them, but we can ignore this\) there may be a loop

point 3 would be that there could be an arbitrarily long sequence of calls A -&gt; B -&gt; C -&gt; D -&gt; ... -&gt; Zn where n is an arbitrarily large integer

I'm going to assert that point 3 is irrelevant because without TCO this would blow the stack and even with TCO, we're not assuming an tail recursion \(or otherwise we'd be up in point 2 where we have "loop \(ie iteration\) === tail recursion"\)

so, we have point 1 and point 2 to deal with, and if there is any dispute or confusion about my definition for them, we should nail that down first

if we're good on those definitions, then I just need two concepts to turn point 1 and point 2 into a "continuation-oriented" computation that eliminates the stack

I'm not asserting that's universally good or relevent to every problem, and recall that I'm on a rampage to stamp out "general purpose" bullshit

Brian Shirai @brixen Nov 12 2017 11:16

so, for point 1, I can conceive of a mechanism to allow the stack to unwind naturally but making the IO mechanism aware that an Actor instance, Ax, called the IO function, and if it will block, the IO can instead return immediately \(possibly some sentinel value\) and arrange for an "IO completion" message to be sent to that Actor, and all the Actor needs to do is have a bit of code designated to handle that message

for point 2, I can conceive of a mechanism \(perhaps not unlike a lazy iterator\) that would convert an explicit loop \(iteration\) into a sequence of messages that can be consumed and processed without retaining the stack as a temporary state store

now, what mechanisms for 1 and 2 look like precisely, I don't entirely know, but I have an inkling

I'm guessing this will be controversial and a bunch of people will call it shit and Haskell IO monads are way better and blah blah blah, but I'm pretty sure this would be a really useful thing to have handy

and if you get a chance to watch that Sussman video up there, you'll see the part where he talks about a bunch of computation models mixing in a program to do special purpose things

and that is really where I'm trying to get to

Brian Shirai @brixen Nov 12 2017 11:25

now, all that said, here's another possibility, with zero stack switching magic \(which I'm 100% going to not allow because how am I going to do that on eg WebAssembly\), we could make the interpreter for these Actors stackless

there's no reason we couldn't do that, and it's a pretty decent solution to the problem of wanting some linear looking A -&gt; B -&gt; C -&gt; IO with explicit looping

and I'm not opposed to doing that, but I question whether we need to do that to have a decent enough system

and here's the biggest thing: can we make this infrastructure suitable for a better language abstraction, and then just use that in these places instead of Ruby, all while mixing in with the rest of the Ruby stuff around it

Chuck Remes @chuckremes Nov 14 2017 13:31

Reading whenever I have a spare moment. Brute force, man!

Interesting comment by Kay at the 26m mark. He thinks that we might have gone the wrong direction by making objects too simple. Maybe we’d be better off making them more complex and capable. Interesting thought that goes against the accepted wisdom \(single responsibility principle and such\).

Brian Shirai @brixen Nov 14 2017 13:49

yes, that part is super interesting, and I'm pretty sure microservices are the capable, properly sized objects

Chuck Remes @chuckremes Nov 14 2017 13:51

Right. So if you look at OOP then the objects we are creating are really the “atoms” of a system. Combining them together we may get some clever “molecules” that are useful. Combining those molecules together into an entire system is where it all gets super interesting. Maybe microservices are the molecules?

Brian Shirai @brixen Nov 14 2017 13:51

but they require a notable shift in at least 3 areas: 1. routing \(and reverse proxies\) become "interstitial"; 2. communication \(eg nanomsg\) become a boundary \(every boundary is both a limit and a contact surface\); and 3. libraries are going to fade and be replaced by services

microservices are cells

Chuck Remes @chuckremes Nov 14 2017 13:52

Sure puts a different twist on message passing. Cells do chemical signaling between each other. They don’t pass other cells between themselves \(where in computing the cell would be a complex object\).

Brian Shirai @brixen Nov 14 2017 13:52

yep. messaging should be more primitive structures than objects

Chuck Remes @chuckremes Nov 14 2017 13:52

yep +1

Brian Shirai @brixen Nov 14 2017 13:52

again, this is my "objects are for interaction; functions are for data"

Chuck Remes @chuckremes Nov 14 2017 13:52

DTOs \(data transfer objects\)

Brian Shirai @brixen Nov 14 2017 13:53

aka "data"

Chuck Remes @chuckremes Nov 14 2017 13:53

!

Chuck Remes @chuckremes Nov 14 2017 14:05

Alan Kay just broke my brain. Something he said in the last few minutes of his talk has made me understand @brixen’s plan for Rubinius \(as a platform\). If you can find the right point of view \(outlook\) to describe a solution to a problem, it’s worth 80 IQ points. By allowing multiple different languages to run on the Rubinius platform, you can pick the language that best describes the problem’s solution. The optimization allowed by selecting the right language \(outlook\) can be a huge benefit.

What do I win?

Domain Specific Languages for everything you do. Seamless interoperation between them on the rubinius platform.

That was a very long talk with quite a bit of preamble. The meat didn’t land until the last 8-10 minutes of the talk. Cool.

Brian Shirai @brixen Nov 14 2017 14:46

@chuckremes awesome!

you win an all-expenses-paid pass to your future success and \(hopefully\) extraordinarily reduced frustration building applications

@chuckremes also, there are other platforms that enable this, but where Rubinius contrasts, I think, is in making it a first-class platform concern that all those languages \(and their authors\) are supported, rather than the \(seemingly\) indifferent attitude other platforms express, or even an hostile one toward languages that are not "the one true language™"

Chuck Remes @chuckremes Nov 14 2017 15:46

I’m interested to see how we can pull this off. Won’t be easy.

Brian Shirai @brixen Nov 14 2017 16:11

the easy things are not the ones worth doing

Chuck Remes @chuckremes Nov 15 2017 11:05

If we build in a socket type like zeromq or nanomsg, then we’ll want this:

[https://github.com/zeromq/zyre](https://github.com/zeromq/zyre)

Chuck Remes @chuckremes Nov 15 2017 15:28

How’s this for a more sane IO inheritance structure?

[https://gist.github.com/chuckremes/db3ea1a8c4a99b0d460fd00d6a13102b](https://gist.github.com/chuckremes/db3ea1a8c4a99b0d460fd00d6a13102b)

An AsyncIO class would be similar/same and would probably inherit from the SyncIO::Block, SyncIO::FCNTL, SyncIO::Stat, and SyncIO::Config classes.

Chuck Remes @chuckremes Nov 15 2017 15:54

Noodling around some ideas on handling Encodings. After pondering this for a while, it seems like a reasonable path forward is to do something like this:

[https://gist.github.com/chuckremes/0e3c9aeb04e01b5edd5f5795a83156eb](https://gist.github.com/chuckremes/0e3c9aeb04e01b5edd5f5795a83156eb)

To add to this, all IO \(whether sync or async\) works with 8-bit bytes. It has no notion of read\_char which should really just be read\_byte. If you want to read a char, that requires some kind of transcoding step. So, wrap the IO up in a transcoder and read/write chars all day long.

So you only pay the cost of the transcoding if you need it. If you are just working with binary data, no encoding toll is charged.

Still working on my API docs for these new IO classes. Will share soon.

Brian Shirai @brixen Nov 15 2017 22:49

@chuckremes interesting RE classes. A couple ideas: perhaps IO::Sync, IO::Async? Then unless there are significant reasons to have Config, FCNTL, Stat be different, those could be IO::Config, etc? \(are there reasons they differ by sync vs async?\) Are the Block, Stream just namespaces or superclasses or namespace + mixin?

also, out of curiosity, have you looked at Plan9 APIs? Here's the doc for net stuff [https://plan9.io/sys/doc/net/net.pdf](https://plan9.io/sys/doc/net/net.pdf)

docs page [https://plan9.io/sys/doc/](https://plan9.io/sys/doc/)

not suggesting it's necessarily a good template, but I think it's interesting reading about what they tried and what they settled on

on another note, this is some very interesting stuff [http://sysml.neclab.eu/projects/lightvm](http://sysml.neclab.eu/projects/lightvm)

Chuck Remes @chuckremes Nov 16 2017 07:58

@brixen IO::Sync and IO::Async are good ideas. I avoided the IO namespace because I didn’t want to confuse it with Ruby’s current IO. Once I finish my experimentation, that top-level namespace can be renamed to whatever.

Regarding IO::Config, that class will probably be abstract with concrete classes for each of the Block and Stream namespaces. For example, the Config object for “opening” a Socket will be different from opening a regular file. There is no notion of \(for example\) O\_CREAT or O\_TRUNC for a Socket but those modes do make sense for a File.

Similarly, for the Sync namespace the Config object will not allow setting O\_NONBLOCK. It would be nonsensical. For Async, setting that mode will be a no op since it is already implied for all FDs.

Anyway, these ideas are still in early stages. The exercise of writing the docs first has been extremely useful.

I’ll be sure to read through the Plan9 stuff. I am happy to steal good ideas from anywhere!

Chuck Remes @chuckremes Nov 16 2017 08:39

@brixen I read the Plan9 network doc. The API is very similar to BSD UNIX. They renamed a few things like announce replaces bind. But they still use listen and accept for handling incoming connections. Was there some particular aspect that you thought might be useful in that doc?

Brian Shirai @brixen Nov 16 2017 11:37

@chuckremes that all makes sense, and no, there was nothing in particular about Plan9, but a few people whose perspectives I value have suggested looking at its design and choices over the years

one thing I'd suggest class wise is avoiding "abstract" base classes and use mixins instead if that works

abstract classes are one of the scourges of OOP

Brian Shirai @brixen Nov 16 2017 11:43

@chuckremes oh, two things I do think are interesting in the Net doc are the conscious attention to scale / distance in the interconnects discussion and the IL protocol tradeoffs, and their dissatisfaction with how they were able to implement Streams

Chuck Remes @chuckremes Nov 16 2017 12:24

@brixen Regarding abstract, I use that term to mean that it’s nonsensical to instantiate it by itself. It might contain concrete methods shared amongst all subclasses, but certain methods are only implemented by the subclasses. I know that sometimes “abstract” means that it defines all methods but they raise NotImplementedError \(or similar\) to force subclasses to implement them. If that’s the case, I agree that is useless and a scourge!

Anyway, lots of ways to skin that cat. Maybe there will be mixins. I don’t know until the code gets written and the specs are finished. I’ve given up on marrying myself to any specific design too early… this is just a rough sketch so I have something to start with.

RE: Plan9, the IL protocol was of interest. Same with the 9P protocol. I looked into 9P a little bit because it could potentially be useful as an inspiration for a protocol to help with distributed object communication.

As for IL, I haven’t beeen able to find much on it.

Chuck Remes @chuckremes Nov 18 2017 17:08

@brixen Had some thoughts while walking the dogs. The IO work I’m doing is hopefully going to be the basis for some distributed messaging platform. I was thinking about message passing and how right now we \(want to\) assume that method signatures don’t change.

But they do. All the time. Version 1.0, 1.3, 1.5, 2.3, etc of the “api” and most of those versions have to maintain a ton of old “signatures” to maintain backward compatibility.

So.

I know Ruby has the method\_missing mechanism from Smalltalk. Can’t we do better?

For example, can we do method matching on the method name but provide a catch-all for those signatures we don’t understand? I’m thinking that foo\(arg1, arg2\) becomes foo\(arg1-different, arg2-different, arg3\) with a foo\(\*\*\*\) to match any messages where the method name matches but the args don’t. We can now have a “method\_missing” specific to each method\_name. Perhaps that fallback massages the old args into something the newer methods understand, or is passes the message to a “legacy object” for processing, or it rejects it. Who knows. Just spit ballin’ here.

Chuck Remes @chuckremes Nov 18 2017 18:16

method\_missing is just too blunt. Need something more fine-grained.

Zak Storer @zacts Nov 18 2017 18:57

@brixen interestingly I never realized this re: bundler

so cool!

Brian Shirai @brixen Nov 18 2017 21:00

@chuckremes I think with multi-methods you get a per-method method\_missing-ish for free

class A

defm m\(a, b\)

end

defm m\(a, b, c\)

end

defm m\(\*, \*\*\)

end

end

I'm not a fan of versions, as you can see from my revision of the Rubinius versioning scheme

but with multi-methods and the "behaviors" \(type-ish\) facility that I described in the post about Rubinius functions, I think a developer has the tools to work with an evolving API

Brian Shirai @brixen Nov 18 2017 21:54

another great place for defm in the Ruby core library are methods like String\#suffix?, which is defined with a \*args parameter but most often called with a single argument

the method has to assume it will be called with an Array, which is pretty silly

Chuck Remes @chuckremes Nov 19 2017 08:34

@brixen Is defm intended for method overloading? I assume so from your response above, but sometimes my assumptions are off. I do think Ruby needs method overloading… it would clean up apis so nicely. But those discussions always rapidly devolve into a discussion of method typing and things go off the rails from there.

So I’m working on a ‘proof of concept’ gem called ruby-io. My intention is to create sync and async versions of the open command along with all of the components that are needed to support it \(Mode, Flags, FCNTL, and a few other classes, the Fiber Scheduler, the IO Loop, the IO Thread Pool, etc.\). It’s a lot of work to get the infrastructure into place to support this one command. After getting open to work, I then want to add listen and accept for Sockets to complete the PoC. Then adding more commands to the File and Socket classes should be fairly easy. I expect to do a lot of refactoring before I make my first push to the github repository.

Setting a deadline for myself in public… first push with a semi-working gem by this Monday evening \(tomorrow\).

Brian Shirai @brixen Nov 19 2017 11:07

@chuckremes toward the bottom of [https://medium.com/@rubinius/rubinius-takes-the-fun-out-of-ruby-21db64ce87a6](https://medium.com/@rubinius/rubinius-takes-the-fun-out-of-ruby-21db64ce87a6)

also, the Rubinius X ideas are still valid, just not doing them in a separate system [https://github.com/rubinius/rubinius-x/blob/gh-pages/public/index.html](https://github.com/rubinius/rubinius-x/blob/gh-pages/public/index.html)

ruby-io sounds like a great approach. I'm planning to work on the Actor, defm and fun stuff after this ridiculously long foray into Ruby code loading \(again\)

I'm stoked, though, that the CodeDB can accommodate source, AST, LLVM IR, machine code without any fundamental changes

Brian Shirai @brixen Nov 19 2017 11:12

just a matter of spitting those blobs of bytes into there and interpreting them on the way out

Brian Shirai @brixen Nov 19 2017 12:22

the fundamental argument I'm making is that IoT is not going to be driven by people installing more computers, but rather by people installing more things that have computers, and that compute is going to be available, so we need to be able to schedule compute onto a network of heterogeneous devices

essentially, what we are doing in cutting edge data centers right now is going to be radically generalized and distributed in the future

@chuckremes btw, what do you think of re-introducing a stackless interpreter to support Actors that want to use a "threaded code" style instead of a "message passing" style?

they'd still need to use the async classes but they would explicitly use a "stack abstraction"

Chuck Remes @chuckremes Nov 19 2017 12:33

@brixen I don’t understand what you mean by a “threaded code” style. In a “message passing” style the messages will be popped off a mailbox in some kind of event loop. What would a “threaded style” look like?

Brian Shirai @brixen Nov 19 2017 13:37

@chuckremes more like normal code where a series of method calls deepen the stack, a call to a "blocking" \(eg IO, timer\) API is made, and the code appears to "wait" there, until the blocking call completes, then the code continues

underneath, these blocking calls are async and cause the Actor to be swapped off the executing Thread until the completion event, in which case it's available to be scheduled again, and the code continues from where it was waiting

essentially, I'm envisioning three separate execution models: 1. actual blocking calls cause the Actor and the Thread it's scheduled on to block; 2. message passing Actor code that uses a regular stack-based interpreter, but uses a sort of "tail call" / "continuation style" to turn a. IO, timer, etc calls into messages, and b. turns "looping" into messages; 3. stack-abstraction-based Actor code that uses async infrastructure that coordinates with a stackless interpreter

Chuck Remes @chuckremes Nov 19 2017 13:40

@brixen Yes, that would be interesting to see. We discussed this last week \(or 2 weeks ago?\). So yeah, go for it. I would prefer “threaded” to “message passing” if I’m understanding this right.

Brian Shirai @brixen Nov 19 2017 13:42

yeah, it's an interesting space, I get your preference for "threaded" code, but I also think that much existing code looks like that because we don't have 1. nice async APIs; and 2. crisp message facilities

Chuck Remes @chuckremes Nov 19 2017 13:42

Agreed. Time to fix that, eh?

Brian Shirai @brixen Nov 19 2017 13:42

anyway, basic point is that we don't have to suffer difficult code because there's really no reason to insist on one execution style

Brian Shirai @brixen Nov 24 2017 23:36

turns out writing back to the CodeDB is a tad bit complicated

Brian Shirai @brixen Nov 25 2017 12:03

figuring out how to initialize the whole system and then tear it down in the proper sequence to avoid use-after-free is definitely not trivial

I want a TLA+ model for this

Brian Shirai @brixen Nov 25 2017 13:17

hm, so this may be something pretty useful actually, re-thinking CodeDB as a "simple" cache layer on top of IPFS

I'll have to look into that

Brian Shirai @brixen Nov 25 2017 13:31

to elaborate slightly, the point of the cache is to pull resources from IPFS into a single flat file that can easily be mmap'd for runtime performance

the initial pull into the cache may be higher latency, but that avoids pulling a ton of stuff, and would support easy \(drop a single file\) cache purge

Chuck Remes @chuckremes Nov 25 2017 14:02

@brixen And rubygems could “register” LLVM IR \(or machine code or whatever\) with an IPFS address and upload it there. Then when Rubinius runs it could just pull this data down instead of regenerating it \(e.g. an embedded platform doesn’t have the horsepower or memory to JIT\).

Brian Shirai @brixen Nov 25 2017 14:02

BAM you, my friend, are the winner

this is exactly what the Rubinius platform will do with ALL code

Chuck Remes @chuckremes Dec 01 2017 14:46

Just learned about the difference between semicoroutines and coroutines. Also learned that the initial Fiber implementation was as a semicoroutine \(strict 1:1 relationship between fibers where the fiber that resumed another fiber must eventually regain control via Fiber.yield\). Then learned that Fibers got the full upgrade to coroutine with the addition of Fiber\#transfer. Never understood what that was for before, but now I get it. And I think I need it.

Brian Shirai @brixen Dec 01 2017 14:50

Fibers just suck, sorry, not sorry

le'me finish this build thing so I can add Actors

Chuck Remes @chuckremes Dec 01 2017 14:50

I don’t think Fibers suck. Perhaps Ruby’s fibers suck but the concept in general is solid.

Brian Shirai @brixen Dec 01 2017 14:51

I don't think the concept is particularly useful or important when we have the concept of Actor

the idea of fiddling with something like Fiber\#transfer is pure silliness

Chuck Remes @chuckremes Dec 01 2017 14:52

Such is the life of a library maintainer. We deal with the hard stuff so the average programmer doesn’t have to.

Brian Shirai @brixen Dec 01 2017 14:53

however, given that we have Fibers, when we have decent Actors you can write something using each and see what it looks like

Brian Shirai @brixen Dec 04 2017 20:49

and we basically have 3 mechanisms we can use overall for Actors: 1. let it consume the thread stack and block, spinning up new threads as needed until the resource for threads is exhausted \(ie you create 2500 on macOS\); 2. use messages by turning IO and looping into messages; 3. use a stackless interpreter to be able to easily multiplex onto a thread

Brian Shirai @brixen Dec 05 2017 20:04

@chuckremes so, here is where I throw a wrench in your class designing: I need \(an\) IO to be agnostic to whether it will complete synchronously or asynchronously

the reason is, that detail should not leak out of the Actor implementation

I want to be able to dynamically chose to execute a single message processing instance arbitrarily, and if the IO is immediately available, it should complete, if it's not, it could 1. block; 2. suspend

it being the implementation at that moment in time, not even the "actor" \(since again, this should not leak out\)

anyway, I'd far rather push decent Actors forward on Rubinius than fuck with anything in Ruby land, other than the basic Ruby support that exists and is improving as a matter of the machine improving

the best-effort message delivery, concurrent processing of messages \(ie pipelining\), and the fact that an Actor does not "have" a Thread, Fiber, Mailbox, Channel, etc, is notable

Chuck Remes @chuckremes Dec 06 2017 08:07

I want to be able to dynamically chose to execute a single message processing instance arbitrarily, and if the IO is immediately available, it should complete, if it's not, it could 1. block; 2. suspend

So here are some of my comments.

One, sending messages TO actors is by definition an asynchronous mechanism. If IO is just another actor, then every message sent to it will be async anyway. Whether or not it can complete immediately is irrelevant.

Two, we cannot trust the underlying OS to tell us if an IO operation will complete successfully or block. There are myriad cases where a file descriptor can be marked as “ready” to read, but then when you attempt the read operation it blocks. If we can’t trust the OS, then the runtime cannot allow the programmer to decide if a single message should be sync or async. The “sync” choice could block even if the runtime thinks it should complete immediately.

Chuck Remes @chuckremes Dec 06 2017 08:13

Three, message pipelining for Actors \(e.g. ATOM mode in Celluloid\) only makes sense for “functional” or stateless Actors. If the Actor maintains state, then no pipelining can or should occur. This is clear from Hewitt’s writings and videos. Here’s a link to an email thread on the celluloid ML where I covered this topic in depth while trying to get thread pooling to work for celluloid \(spoiler, it’s impossible\).

[https://groups.google.com/forum/\#!msg/celluloid-ruby/htqsuhCLXRw/1NI6C-07BQAJ](https://groups.google.com/forum/#!msg/celluloid-ruby/htqsuhCLXRw/1NI6C-07BQAJ)

Four, this request is very similar to how async-await is implemented for C\#. I can scare up some links if you need them, but the way it’s defined is that the “async” method will execute synchronously until it hits a blocking operation. This includes executing “await”-able methods if their results are immediately available. It apparently takes a lot of compiler support to make that work. However, it results in much improved performance since a synchronous operation is faster than an async operation \(less work to shunt the async op off to a background thread, etc\).

Chuck Remes @chuckremes Dec 06 2017 08:18

Final comment… this is definitely doable \(see comment 4\) but compiler support is required to make it fast.

My original approach to the IO rewrite was to make everything appear synchronous to the programmer even if it would be handled asynchronously behind the scenes. You requested that both Sync and Async be explicitly supported so the programmer could choose performance \(sync\) over scale \(async\). I see that you are still requesting that but want the API to be unchanged. I think this is doable by providing a keyword arg on every IO method, something like nonblocking: false or nonblocking: true. Make it false by default which will cause the library to choose the synchronous path. If nonblocking is true, then it chooses async path. But programmer is presented with same API regardless.

Brian Shirai @brixen Dec 06 2017 11:04

I have Gul Agha's book [https://www.amazon.com/Actors-Concurrent-Computation-Distributed-Systems/dp/026251141X/](https://www.amazon.com/Actors-Concurrent-Computation-Distributed-Systems/dp/026251141X/)

it is definitely pretty theoretical

basically, what I'm saying is that I'm not making any assumptions about how to implement Actors but I'm definitely not limiting myself to Ruby Threads or Fibers \(I wouldn't use Fibers for anything\) or IO

the reason I want IO to have explicit sync and async is that sync implies blocking, not that blocking will necessarily occur

but if someone wanted to open an fd in a Thread and read/write to it endlessly, there should be nothing "async" interferring and that Thread should get the theoretically maximum throughput the OS would allow in that particular instance given whatever else is running

the reason to have a version of IO that just does reads and writes without any designation of sync or async is because that is a detail that absolutely should not leak to the Actors

the most promising, and interesting to me, thing about Actor Model is that it's a physics model, not a mathematics model

a great deal of harm has been done since the proposal of the lambda calculus to model computation

it's a platonic ideal sort of thing

Brian Shirai @brixen Dec 06 2017 11:09

hence necessitating the note on distributed computing, which is literally nothing more than multiple restatements of the fact that the speed of light has a limit

the indeterminism, best effort delivery, and out of order delivery are all physical realities

going even further, the inconsistency robustness is hugely important and it's going to take over, absolutely no doubt about it

ACID and RDBMS are another mathematical model that is simply not reflective of physics

anyway, I'd summarize that the details of the Actor model actually make it much easier to implement than the fanciful abstractions that people have tried applying to computation

but that doesn't mean that people will go use it

we're still using location based addressing on the Internet and building ad hoc content addressing systems \(like CDN's\) instead of getting behind something like IPFS

Chuck Remes @chuckremes Dec 06 2017 11:14

Let’s put aside Actors for a moment… I do want to get back to that in a bit. I want to drill down on what you are looking for in IO. So, sounds like you want file descriptors to be exposed through an API as a first step. Presumably you also want FDs to be wrapped in an object so a programmer can interact with them sanely. When interacting via the object API, should all messages default to sync unless explicitly marked as async or do you want the IO library itself to be smart enough to choose one or the other?

Brian Shirai @brixen Dec 06 2017 11:14

just found this last night [https://twitter.com/carlhewitt/status/39727943148244992?lang=en](https://twitter.com/carlhewitt/status/39727943148244992?lang=en)

Chuck Remes @chuckremes Dec 06 2017 11:17

example: io.read\(offset: 17, length: 30, tobuffer: buffer\) \# sync

versus

io.read\(offset: 17, length: 30, tobuffer: buffer, nonblocking: true\) \# async

Brian Shirai @brixen Dec 06 2017 11:18

for IO: 1. yes, fd should be available \(for eg FFI\); 2. for the object wrapped versions, I think there should be different classes that can be used explicitly, so SyncIO\#read implies blocking, ASyncIO\#read returns a completion object \(promise?\) and explicitly does not block \(ie does not attempt the IO operation in that some thread\); 3. an interface \(probably OO, but I'm not insisting on it\) that just has eg IO\#read and whether it blocks on not, and what exactly happens is determined by the VM consistent with the Actor semantics, which means that it could literally read and block, then deliver the message when it was done, or create a completion object that delivers the message

Chuck Remes @chuckremes Dec 06 2017 11:18

same api to read but programmer can get async behavior when requested.

Brian Shirai @brixen Dec 06 2017 11:19

the reason I think the classes should be different for Async and Sync is that there is different machinery required

SyncIO wouldn't have any sort of completion object, etc

so, the "designate at method call" approach seems to me to be totally unworkable

Chuck Remes @chuckremes Dec 06 2017 11:20

I guess I’m asking if you want to take it up one more level of abstraction. That is, there is an IO class which has private internal classes Sync and Async which it can use. However, IO class provides singular API for interaction and it delegates to its Sync/Async private classes at runtime.

Brian Shirai @brixen Dec 06 2017 11:20

further, as I pull these primitives into the instruction set, I want to be able to disallow certain code from running based on the classes of instructions it uses

so, the last thing you said, right now I'm thinking about that IO class in terms of Actor semantics, while thinking about the ASyncIO and SyncIO in terms of conventional threads and concurrency like futures and promises

Chuck Remes @chuckremes Dec 06 2017 11:23

Sorry, that’s not making it clearer for me. Will the programmer see a Promise/Future returned or is that an implementation detail that the IO library will manage?

Brian Shirai @brixen Dec 06 2017 11:26

hm, ignore that IO for now, I'm only thinking about it in terms of Actors so it's going to be confusing otherwise

Chuck Remes @chuckremes Dec 06 2017 11:26

[https://gist.github.com/chuckremes/ab075d21917441f0cc85c35d19becb20](https://gist.github.com/chuckremes/ab075d21917441f0cc85c35d19becb20)

^^ Both block until the read is complete. One is a sync read, the other is async.

Brian Shirai @brixen Dec 06 2017 11:27

something like that could be possible, similar to a Maybe in Haskell, where you could test which resulted \(data vs a promise\)

so, what your gist looks like to me is that the client code would think the IO call was sync no matter what \(ie that code would not progress\) but it could possible get swapped off a Thread waiting for IO to complete, and then it would continue

that's the Go model, as best I understand

and it's not terrible, but I'm not sure it's needed because we have proper Threads that can be used with blocking IO

Chuck Remes @chuckremes Dec 06 2017 11:29

Right, that’s the intention. From end-user programmer perspective, they write code in synchronous/procedural fashion.

Brian Shirai @brixen Dec 06 2017 11:29

and the ASync would be intended for code that was structured to work with futures and promises

for example, I want to make 10 calls to other services, wait for them to complete for a certain amount of time, processing results as they are available

Chuck Remes @chuckremes Dec 06 2017 11:30

Sure, we do have proper Threads but it doesn’t solve C10k problem. With advent of 64+ cores, problem gets worse. Can’t assume 1 thread per IO op.

Brian Shirai @brixen Dec 06 2017 11:30

that model is really useful and without proper support, you have to make it up yourself every time with explicit Threads usage

right, these are two different models

the C10K problem needs something like Actors or AsyncIO, but it's entirely possible that you want a process that is a producer or consumer and they just process data as fast as it flows to it

these are different things that need different support, which is why I'm constantly saying "everything cannot be an X"

whether that X is an object, a function, sync IO, etc

the attempt to implement C10K problem with a Thread per connection is completely wrong-headed

the attempt to implement a data processor for a single stream of data with a bunch of unnecessary AsyncIo is completely wrong-headed

Chuck Remes @chuckremes Dec 06 2017 11:34

Yes, AsyncIO for C10k problem, SyncIO for single stream of data.

Brian Shirai @brixen Dec 06 2017 11:34

and again, the reason I do not want SyncIO implemented as a "blocking" layer on top of AsyncIO

Chuck Remes @chuckremes Dec 06 2017 11:34

I’m not suggesting we do.

Here’s where I am still confused on your postion...

Brian Shirai @brixen Dec 06 2017 11:35

now, with Actors, because these semantics cannot leak \(if implemented correctly\), it should be possible to pick one or the other in a context aware way, but the reason I think that will work is because the structure of the Actors means that the code will be very different than conventional \(possibly threaded\) code that we see today

Chuck Remes @chuckremes Dec 06 2017 11:35

Are we letting the programmer explicitly choose Sync or Async IO? If we are, do you intend for the IO APIs to be the same wherein Sync returns data and Async returns a Future?

Brian Shirai @brixen Dec 06 2017 11:36

yes, that's what I'm thinking

the code is structured very differently

Chuck Remes @chuckremes Dec 06 2017 11:36

Why expose the Future to the end-user programmer? Isn’t that leaking the abstraction out?

Brian Shirai @brixen Dec 06 2017 11:36

not to me, it isn't

the code is explicitly structured to work with futures and promises

Chuck Remes @chuckremes Dec 06 2017 11:37

hmmm, okay

Brian Shirai @brixen Dec 06 2017 11:38

again, the example of requesting data from 10 services when a service gets a request: 1. it's going to fire those 10 and get back 10 promises; 2. it's going to process data in the order the promises complete; 3. at some point to respect it's service level agreement, it's going to abandon not completed promises and return its best effort

Chuck Remes @chuckremes Dec 06 2017 11:39

Will failed promises/futures raise exceptions?

Brian Shirai @brixen Dec 06 2017 11:40

I'm open to arguments, I don't have a strong opinion, other than the error shit that Go does I'm totally not ok with

at least, not at this high level

so, ignore the IO thing, just consider the ASync, Sync cases for now

Chuck Remes @chuckremes Dec 06 2017 11:40

ok, listening

Brian Shirai @brixen Dec 06 2017 11:41

I'm going to take a look at ActorScript if I can to get an idea how something like IO is implemented

Chuck Remes @chuckremes Dec 06 2017 11:41

I’ll look too.

Brian Shirai @brixen Dec 06 2017 11:41

I mean, just focus on Sync, ASync for now

Chuck Remes @chuckremes Dec 06 2017 11:42

To me, Sync/Async is the whole ball game. IO is just one implementation.

Brian Shirai @brixen Dec 06 2017 11:42

ie Sync really should explicitly imply blocking, the client code will not progress until the data is returned, whereas ASync IO never blocks and expects the client code to work with promises

I think there are three games: 1. Sync IO and conventional threads; 2. Async IO, explicit promises, and optionally threads; 3. Actors

I'll keep you posted on 3

Chuck Remes @chuckremes Dec 06 2017 11:43

brb

Brian Shirai @brixen Dec 06 2017 11:43

moi aussi

Chuck Remes @chuckremes Dec 06 2017 11:50

back...

I think Sync and Async are the only 2 games in town. Actor is an implementation detail.

Whether we are doing a blocking IO operation or a long-running computation, we can wrap either in a Sync or Async API. We’re expecting an answer back. In Sync case, we get the data directly. In Async case, we get a Future that we can resolve when we need to access the value.

A Future is just a kind of Actor \(as mentioned multiple times by Hewitt\). A Future is how an Actor can talk to itself without deadlocking, for example.

Chuck Remes @chuckremes Dec 06 2017 12:09

Where the hell is the source code to ActorScript?

Hmmm, no implementation of ActorScript exists in the wild that anyone can find. Old thread on this: [http://erlang.org/pipermail/erlang-questions/2014-June/079906.html](http://erlang.org/pipermail/erlang-questions/2014-June/079906.html)

[http://carlhewitt.info](http://carlhewitt.info) also doesn’t exist anymore

vapor

Brian Shirai @brixen Dec 06 2017 16:14

all the more reason to implement Actors, @chuckremes

I think getting Sync and Async working well is going to illuminate a lot of the important aspecs of implementing Actors well, but Actors won't just be a composition of Sync and Async under the covers \(that's my hypothesis and I'm pretty confident about it\)

Brian Shirai @brixen Dec 06 2017 16:28

the Actor Model explicitly defines a boundary between what is not an Actor and what could be, which disavows the typical and ludicrous "everything is an X" or "turtles all the way down"

cells are not built of cells, organs are not built of organs, etc

I see no reason a thread, address, or message would be an Actor

and if you think about pipelining for a moment, imagine that \(going back to Alan Kay where you don't have "instance variables with getter asd setters"\) you do some data flow analysis that identifies data dependencies and you have two independent pieces of data that are altered in the process of sending messages

image msg m modifies state a and msg n modies state b

the actor starts processing m and sends n

msg n starts processing

there's no problem and it doesn't matter when n starts or finishes because a and b are independent

Brian Shirai @brixen Dec 06 2017 16:34

there is also no problem with this code running on separate threads on a typical multicore CPU and doesn't require any data synchronization

this cold has totally laid me out for two days, going back to bed

Chuck Remes @chuckremes Dec 06 2017 17:18

If that kind of data flow analysis is cheap to do at compile time, then sure you could run those things in parallel \(ATOM mode\). If that analysis isn’t cheap, then I stand by my earlier assertion that pipelining only makes sense for functional Actors.

As for the “turtles all the way down” issue, I agree. Hewitt’s papers sometimes make it seem as though creating Actors on top of “primitives” like queues and Threads is wrong. Just an impression I get.

Chuck Remes @chuckremes Dec 11 2017 12:46

@brixen What’s your opinion of the Maybe Monad where a function/method returns a wrapped value?

Brian Shirai @brixen Dec 11 2017 12:47

@chuckremes very good question, but instead of giving you my thoughts on it, consider reading Gilad Bracha's thoughts, and then we can discuss  [https://gbracha.blogspot.com/2011/01/maybe-monads-might-not-matter.html](https://gbracha.blogspot.com/2011/01/maybe-monads-might-not-matter.html)

Chuck Remes @chuckremes Dec 11 2017 12:48

Will do. Back later.

Chuck Remes @chuckremes Dec 11 2017 14:46

@brixen Read it. I think that article is addressing a much broader topic that I was trying to focus on. Right now I’m looking at providing two Policies for returning results from IO functions. One, is to return the a\) return code, b\) errno, and c\) other function-specific effects. The programmer then needs to decide what to do based on return code. Two, raise an exception if the function fails or use the return values directly. Programmers know how to deal with this too.

I was curious about the “Maybe Monad” \(or the Option pattern?\) since it can return a tuple: \[:ok, value\] OR \[:error, value\]

That could be a third type of Policy.

Anyway, I’m probably getting ahead of myself here. Just wanted your opinion on that particular pattern.

Brian Shirai @brixen Dec 11 2017 15:06

@chuckremes I think the \[:ok, value\] pattern is for languages with poor support for types or poor support for exceptions \(or both\)

now, it's possible you'd like the interface to always return an object and allow the client code to conditionally do something, in which case you can return an object that would raise when the result were accessed, but could return other, consistent information, for other attributes

totally spit-balling here, but imagine it was a ReadResult or ReadError and both had a \#size attribute

ReadResult\#size would return the size of the result read, while ReadError\#size would return 0

ReadResult\#data would return bytes where ReadError\#data would raise an exception \(possibly raising itself?!\)

Chuck Remes @chuckremes Dec 11 2017 15:09

Sure, kind of like the NullObject pattern \(or at least that’s what jumped to mind first\).

Brian Shirai @brixen Dec 11 2017 15:09

something like that

the question to me really is, what is the value of allowing the client code to conditionally raise that exception, versus the client code not having that ability

but in every case, the exception object should be full-featured, not the typical, anemic exception we see in Ruby

\(where, eg, people are driven to parsing text returned from caller\(\) to find a line number\)

Chuck Remes @chuckremes Dec 11 2017 15:11

Agreed. Current exceptions are poor. The IO exceptions should provide a lot more insight into the failure \(when possible\).

Brian Shirai @brixen Dec 11 2017 15:11

the one case for the above described approach would be this: you need to do some sort of resource finalization in a context where an ensure block is unweildy

the one case I can think of now

Chuck Remes @chuckremes Dec 11 2017 15:32

Your mentioning of Become the other day reminded me of the Gang of Four definition of the State pattern. I’d like to implement that in this new IO lib. Purpose would be to handle exceptional conditions or even just a plain old close on the file descriptor. Rather than put “state checks” in every method, just swap the internal io object reference to a new object that handles that particular state. The outer io object just delegates to this inner one. Upon a call to close it closes the file descriptor and then swaps the internal reference to a new object that handles a “closed” state.

The GoF state pattern description I believe described this in terms of Become.

Chuck Remes @chuckremes Dec 11 2017 15:42

This notion makes a lot of sense if the IO object is an Actor. When it processes the close messages it tells itself how to handle future messages by “becoming” the EBADF\_IO actor where all calls return EBADF \(or raise depending on the set Policy\).

\(Just thinking out loud here in a rubber duck session now…\)

Brian Shirai @brixen Dec 11 2017 15:43

sure

Chuck Remes @chuckremes Dec 11 2017 15:43

What other IO states are there? Clearly there’s an open state and closed state.

Brian Shirai @brixen Dec 11 2017 15:44

with delegation overhead, you could achieve the first with a wrapper object

you could considered available or exhausted \(eg eof\) states, i suppose

Chuck Remes @chuckremes Dec 11 2017 15:45

If we make the IO lib stateless, we avoid needing a waiting or busy state since we never lock. This presumes the use of pread and pwrite which require a file offset for every operation. Serialization&ordering of writes would be the programmer’s burden.

Chuck Remes @chuckremes Dec 14 2017 14:59

Heh, cool.

BTW, while thinking about actors recently and how a mailbox isn’t an actor, a thread isn’t an actor, etc. I’ve decided to summarize my view of Actors like this:

Chuck Remes @chuckremes Dec 14 2017 15:04

An Actor is a molecule. It is made up of some number of atoms which may or may not include a thread, a mailbox, a queue, etc. If you decompose the Actor molecule, you get the components of an Actor but you do NOT get an Actor. This avoids the turtles-all-the-way-down rabbit hole often referenced by Hewitt.

anyway, seems obvious. whatever.

Chuck Remes @chuckremes Dec 14 2017 15:17

So far this talk by Gul Agha \(at my alma mater UIUC\) is a good fit for me. Practical discussions about how to actually implement these Actor concepts. About halfway through his video.

Brian Shirai @brixen Dec 14 2017 15:18

yeah, that's a great video

and I agree, you can have the components of an Actor in one hand, but they aren't an Actor until they are an ensemble

what's especially interesting to me about this, it's a perspective much like Constructor Theory

Chuck Remes @chuckremes Dec 14 2017 15:19

The fairness issue seems like an easier fix than what is \(so far\) discussed in the video. We already have the operating system pre-empting threads to let others run. Why can’t a userspace thread have its own scheduler and preempt an Actor to allow another one to run. Seems obvious too though now we have potentially layers upon layers of schedulers. Could go down a rabbit hole too.

Brian Shirai @brixen Dec 14 2017 15:20

there's an excellent 15min video of Chiara talking at a conference about extinction and she's noting that when we talk about extinction, for instance, of a particular bacterium, it's not the atoms that go away, but the particular organization of those into a thing \(an organism\) that can replicate

extinction is the end of the ability to replicate

Chuck Remes @chuckremes Dec 14 2017 15:21

yes, good perspective

Brian Shirai @brixen Dec 14 2017 15:21

she then notes that ideas have taken the place of organisms in evolution in that our ideas can be criticized and "made extinct" in place of ourselves, "our ideas die in our place" is a quote she uses but I forget who said that

Chuck Remes @chuckremes Dec 14 2017 15:23

Right, I saw that part of the video \(you shared that maybe a week ago\). Having an idea “die in my place” is certainly preferred over my actual physical death.

Brian Shirai @brixen Dec 14 2017 15:23

as for the fairness thing, I'm still thinking about it, but by considering an Actor to be this boundary around its behaviors, instead of coupled to any particular stack or thread, I'm starting to see how to have concurrency internal to the Actor

Chuck Remes @chuckremes Dec 14 2017 15:24

Okay, like object a sent message dothis and message dothat can run them in parallel if those behaviors don’t share any internal state.

Brian Shirai @brixen Dec 14 2017 15:24

exactly

sometimes just a turn of phrase can open up a whole new vista: in one of Hewitt's videos about the future of intellectual property, he called a actor "a message receiver", and that was a light bulb moment for me

it's one of the Actor axioms \("can create new Actors"\), but just turning that around from "Actor" to "a message receiver" is such an interesting shift in perspective

Chuck Remes @chuckremes Dec 14 2017 15:26

I’m playing with a similar idea right now to support io.each. If the each “behavior” is passed in its args and uses only variables visible to the local method, then you can have any number of threads sharing that same io and running each without them stomping on each other.

Brian Shirai @brixen Dec 14 2017 15:26

now I'm thinking, ok, "what does 'a message receiver' look like?", instead of "what does it look like to deliver a message from Actor A to Actor B?"

Chuck Remes @chuckremes Dec 14 2017 15:26

The main problem is avoiding the shared state of the idea of current offset within a file or other IO mechanism. Anyway, Ruby gets so much wrong here.

It looks like a giant switch \(in C\) statement!

Brian Shirai @brixen Dec 14 2017 15:27

yes, Ruby gets so much wrong s/here// lol

Chuck Remes @chuckremes Dec 14 2017 15:30

If you want to get parallelism from a single Actor, then fronting it with a single queue is the wrong way to go. That enforces a single step-by-step processing of incoming messages. If each behavior \(method\) has its own locus of control, then each method could have its own mailbox \(or queue\).

But again, assumes no shared state, etc

Hmmm, stray thought...

So an actor processes messages and a message MAY cause it to process the next message different \(i.e. become\).

If you allow multiple behaviors to run in parallel, does that mean a message processed by behavior1 can change how behavior2 processes its next message?

I’d assume NOT

It now means that behavior1 can only change how it processes the next message sent to behavior1.

So we’ve now devolved the Actor down to a single method call. If you want to run multiple behaviors in parallel, they need to go into their own actors. Bah humbug.

Brian Shirai @brixen Dec 14 2017 15:36

well, I guess I'm not clear on that then

the axioms say "can designate how to process the next message" but there's nothing that says that message is processed by this behavior

I've been roughly assuming "behavior" == "how to process a message of type T"

I could be wrong about that, so I'll have to dig some more

Chuck Remes @chuckremes Dec 14 2017 15:38

yeah, that is ambiguous hence my worry.

Brian Shirai @brixen Dec 14 2017 15:38

but again, by bringing stuff like "become" to the instruction set, I want to be able to do analysis on the data dependency and operational dependency between "parts" of the Actor

thinking from the perspective of "a message receiver", the incoming message is merely the "invocation" of some computation

hence, I think, Hewitt's insistance that a message can be no more costly as a lower bound than a procedure call

the thing is \(and is mentioned by an attendee in the video above\), there is a cost to consistency and that cost is manifest by "waiting"

ie an operation that is not depending on consistency will "wait" less than an operation that does depend on consistency

the thing I'm still getting my head around is where these costs manifest

and I think Hewitt's point about Actors being a model of computation is that the model must not "add to" the cost of the computation

in other words, it's a model of the computation itself, it is not a simulation \(ie added cost\) of the computation

this may seem like a distinction without a difference, but by Hewitt's insistence on it, I'm pretty certain the distinction has consequences for correctness

Chuck Remes @chuckremes Dec 14 2017 15:44

Have you tried engaging with Hewitt directly on any of this?

Brian Shirai @brixen Dec 14 2017 15:46

hah, funny you should mention that! I'm working on it

Chuck Remes @chuckremes Dec 14 2017 15:47

Cool. Tell him to join us on gitter so I can pester him too \(j/k\)

Brian Shirai @brixen Dec 14 2017 15:47

haha

I can try!

I hope his health is good, I worry he's going to be gone before we make progress on these things he's been talking about

all the website domains he's mention in the past 5 years are gone

some people find his constant laughing in his talks annoying but I find it pretty amusing

I wonder if he drinks whiskey



# The Last Word

Brian Shirai @brixen Feb 05 22:46

@chuckremes "The religious war over connections and connectionless had been at the root of too many disasters." let's see, if we s/connections and connectionless/object-oriented and functional/ or s/.../evented \(ie nonblocking\) and direct IO/ or s/.../static and dynamic typing/ or s/.../interpreted and compiled/ seems like we might be seeing a pattern emerge here 



Brian Shirai @brixen Feb 05 23:48

@chuckremes without getting distracted by Actor details, I'd suggest there are 3 ways we can look at IO: 1. explicit blocking calls, the code makes a "semantically" and physically blocking call and accepts that the stack \(thread\) it's running on is blocked; 2. explicit blocking calls that are non-blocking underneath, the code makes a "semantically" blocking call but underneath, its stack is swapped off the thread \(like Go\); 3. explicitly non-blocking calls with some sort of continuation mechanism

a couple points I'd make: 1. none of these are "general purpose" \(I'll get into that in a sec\); 2. it's critical to keep the pieces in play well separated, there is a stack, a thread, some code, and an IO call

the reason I push back so hard on Fiber being legitimate is that it's an attempt to say "I'm just as stack, look, I got suspended and resumed" but it's a thread and as stack

in case 1, the thread and stack are explictly bound; in case 2, the stack is implicitly bound to the thread; in case 3, it's not entirely clear that there is a thread \(think JS, you cannot touch that "thread" but you can run a bunch if little functions\)

so, in case 3, you basically just have these stack things and you pass the continuation function \(callback hell\) that represents a "stack in waiting" so to speak



Brian Shirai @brixen Feb 05 23:54

case 1 is great when work bounds are clear: I have a set of threads or even a thread pool and set work and there is IO that flows through that, I have nothing else better to do if there's no IO so waiting on it is just fine. This is totally legit

case 2 is great when work is highly heterogeneous and lots of stuff of different sizes is all mixing together. Again, totally legit but different than 1

case 3 could be useful when the work is rather mixed but well defined, and this is where I think the structure of Actors can be extremely useful

and case 3 is where I think we can use an explicit non-blocking mechanism that does not result in call-back hell by leveraging an Actor sending a message when processing a message, turning blocking calls and looping into message sends essentially



Brian Shirai @brixen Feb 05 23:59

now here's the implementation details: case 1 is simple if you've got threads and concurrency \(including potential parallelism\), and we've got that covered \(but needing a better API for IO\); the mechanism for case 2 is stack switching and I'm not going to introduce platform dependent asm-based stack switching so until POSIX or C++40 does, we're limited to stackless interpreter for that

it's case 3 that gets interesting to me, if we implement Actors well and IO well, then we should be able to use normal threads and message loops that essentially trampoline, allowing them to be "switched" on and off a thread using nothing more than a function \(method\) call



Brian Shirai @brixen 00:05

anyway, dumping this here because I'm not going to get to it in the very near future, but it's a constant consideration and I've approached the new lookup, cache, invoke instructions with the intention of being able to execute code stack-based or stackless without needing to change the instruction stream

but you would have to designate the Thread when creating it to be one that executes stackless \(which could be hidden behind some other object facade of some sort, whatever\)

now here's the rub for IO, and something for you to ponder: if the intention is that the same piece of code could execute stack-based or stackless, then if that code makes a semantically blocking call, what is the mechanism that in case 1 makes a physically blocking call and in case 2 causes a stack swap and suspend waiting on an IO completion event?

of course, we could say, if you are going to run this code in a stackless Thread, then it can only use "blocking implemented with non-blocking IO", and I'm pretty much ok with that, except it doesn't work with libraries really \(how do I know what this library is going to do way down in some code path?\)



Brian Shirai @brixen 00:11

we could go the other way and say, if you want to use physically blocking IO, you must do that explicitly, otherwise you get fake blocking IO, and if you use really blocking IO in a stackless thread you just blocked the thread and enjoy your consequences

neither of these two options seem that great to me and I think it's possible to use a uniform API so that a call to a physically blocking IO function and one to a blocking-non-blocking one look the same, and use a late binding mechanism to choose one based on context \(ie the attribute of the Thread the code is running on\)

this makes it possible to have confidence that when my code is explicitly designated \(invoked\) on a stackless Thread, some library that makes a blocking IO call actually makes a blocking-non-blocking IO call, which should be the same semantically but that stack just got switched off the thread

library is none-the-wiser



Brian Shirai @brixen 00:17

note that in both these cases, I'm talking about semantically blocking IO

in case 3, the API of the IO is explicitly non-blocking, and I'm ok building that with the Actor mechanism because I think they go well together, but it's fine to allow people to use it in a callback-hell way, I'm just neither advocating doing so nor that willing to be sympathetic to their pain \(especially with the Actors there for the using\)

but the Actor mechanism doesn't have to use the explicitly non-blocking IO because it works just fine with case 2

so you could code the Actors to make blocking-non-blocking IO and use loops instead of turning those into messages and Actors would run on stackless threads



Brian Shirai @brixen 00:26

let me know if that makes things more clear

if not, I'll draw some pictures 

but I need to spend the limited cycles on the GC and JIT in the near term, so I'm not going to be diving into any code

\(other than possibly spiking the stackless interpreter when I implement the invocation instructions\)





