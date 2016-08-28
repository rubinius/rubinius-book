# Preface

> Nothing pleases people more than to go on thinking what they have always thought, and at the same time imagine that they are thinking something new and daring; it combines the advantage of security with the delight of adventure &mdash; TS Eliot

Nothing in this book is really new. No silver bullets that will make you wildly rich, world famous, or powerful as presidents. Instead, my hope is to change your view of programming.

I want to change two things about how we view programming. First, programming is *too special*. Far too many people are excluded from programming. I want to make programming *mundane*. Second, programming is *too limited*. We have extremely crude tools for programming. I want to make programming a *powerful* tool broadly used to solve problems.

Even worse, much of the conventional knowledge about programming is just wrong. The false fables of programming are the most disturbing.

I've heard all of these: register virtual machines are faster than stack machines; optimizing compilers can't beat hand-tuned assembly language; garbage collection is way more costly than manual memory management; static typing makes programs safer than dynamic typing, reference counting garbage collectors aren't as good as tracing collectors; tracing JITs are better than method JITs; dynamically typed languages are slow; functional programming is better for concurrency than object-oriented programming; mutable state makes concurrency hard; event-oriented programs can handle more connections than programs using threads.

I could go on and on. While some of these assertions may be partially true, or more true in a particular context, they are not absolutely true. They represent a small fraction of the *things we know that just ain't so*.

> It ain't what you don't know that gets you into trouble. It's what you know for sure that just ain't so. &mdash; Mark Twain

Why is there so much misinformation in programming? I'm not sure. I doubt there is necessarily more in programming than in other fields. For a philosophical inquiry into misinformation, I recommend *On Bullshit*, by Harry G. Frankfurt.

> Sometimes the questions are complicated and the answers are simple &mdash; Dr. Seuss

I may not have a satisfying answer for the misinformation programming, but I can offer an antidote: a system that combines the supposed *either A or B* concepts into a comprehensive and integrated whole.

This is a principle that applies widely: When confronted with *A or B*, consider why not *A and B*. In mathematics, there is the concept of a *dual*. For example, in Euclidean geometry, lines are dual to points. Take a theorem and swap lines for points and vice versa and the theorem is still true.

It's not that one *substitutes* for the other. We don't replace lines with points, or vice-versa, eliminating the opposite one. In fact, these concepts are frequently defined in terms of each other. The familiar, "a line is the shortest distance between two points".

We would think it silly to be told in geometry we could have lines or points, but not both. But when we are programming, we tend to accept, as if some law of the universe, that we can have *either* static typing *or* dynamic typing, for example, and even worse, that one is "better" than the other.

When I hear things about programming that are untrue, I notice two sorts of confusion: mistaking things that are actually different as being the same, and things that are the same as being different. I've written down these observations (in somewhat humorous form) as **The Eight Fallacies of Programming**.

1. *Same Scale*
2. *Same Risk*
3. *Same Cost*
4. *Same Granularity*
5. *Same Abstraction*
6. *Same Temporality*
7. *Same Order*
8. *Same Applicability*

What I've realized is that our fascination with "general purpose" programming languages is a vicious circle that feeds these biases *and* is powered by them. If problems are the same scale, then a general approach is likely the most efficient to solve them. And the flip side is true as well: if we have a *general* tool, it should reduce problems to a common solution.

The truth is, the scale of problems varies across an enormous spectrum. If we're going to improve the way we build software, we need a more accurate model of programming. We need to recognize that programming is a behavioral science. It encompasses intentions and values, meaning and purpose, ethics, politics, and economics. It is intensely humanistic.

This is a big change.

> You never change things by fighting the existing reality. To change something, build a new model that makes the existing model obsolete. &mdash; R. Buckminster Fuller

I'm not on a crusade to change your mind. My goal is not to convince someone who loves Haskell that they should love Ruby instead. Or to convince someone who believes Rust will save us from ruin that garbage collection is a critical component of modern software. Not only is that impossible, it's ridiculously presumptuous! I'm not the judge of what ideas you think are important.

That may sound like a contradiction. First I said I hope this book will change your perspective, and then I said I'm not trying to change your mind. What gives?

One aspect of it is intention versus outcome. It is my intention to argue persuasively that a synthesis of apparently opposed ideas is more useful to us. It is my intention to demonstrate this with Rubinius itself, but also with the programs and languages you create. My focus is on my intention, not on the outcome. I can control the former but only influence the latter.

More importantly, I'm assuming you're more interested in learning than you are committed to some, possibly false, notion about how programming works. If I'm wrong about that and you are a devout adherent of some sect of programming dogma, like functional programming being the one true way, I'm not going to be in conflict with you. Maybe you'll hang around or maybe you'll get bored and wander off. I don't need to change your mind.

But if I'm right and you are most interested in learning, I'm going to focus on my intention to provide the best possible environment to support your learning. And now we've cycled back to my intention versus attachment to an outcome.

In other words, come and go as you please. If this is interesting and I can make it more so, let me know. If everything I'm saying is wrong, save your time and ignore it.

> Take what you can use and let the rest go by. &mdash; Ken Kesey

One may think that this should go without saying. As we mature, we learn that the world is wide and varied, and things don't actually reduce to simple categories with clear boundaries. And yet, mature behavior is not what we witness on the Internet every day.

The programming community, if one can reasonably cast such a wide net, is no different. We talk of language wars and platform wars and framework wars. We write volumes, backed up by statistically shaky (or false) data like benchmarks, about how this language is better than that language or this framework is better than that framework. There is one right answer, we have it, and the rest of the Internet better agree.

That sort of behavior is unhelpful and unwelcomed here. In its place, we're going to formulate and support a way to work together that provides expansive room for experimenting with new ideas and minimizes unnecessary conflict. Further, it provides a framework for making decisions and validating results.

> Don't worry about people stealing your ideas. If your ideas are any good, you'll have to ram them down people's throats. &mdash; Howard H. Aiken


## Organization of This Book

*Part I - The Rubinius Language Platform*

* *Introducing Rubinius*
* *Getting Rubinius*
* *Console: Portal to Another World*
* *Atom Terminal: Joining Two Worlds*
* *Instructions: The Essence of Program Behavior*
* *Interpreters: A Program at Play*
* *Parsers: Seeing Trees in a Forest of Characters*
* *Community: Playing Well Together*
* *Compilers: Weaving Webs of Behavior*
* *Scopes: Keeping Things in Tidy Boxes*
* *What, How, Why: Understanding Program Behavior*
* *Garbage Collection: Stay in Touch to Stay Alive*
* *Just-In-Time Compiler: Using the Machine Under the Machine*
* *CodeDB: A Memory for Programs*
* *Debugging the Platform*

*Part II - Understanding Programming Languages*

*Part III - Writing Your Programming Language*

*Part IV - Working in the "Real World"*

*Part V - Rubinius Reference*


## Audience for This Book

This book is written for anyone who is interested in learning about programming languages, including virtual machines, compilers, garbage collectors, and the tools to understand and debug these systems.

The book does not assume any previous experience with systems like these or with programming in general, but it does assume the reader has a willingness to learn and to follow references to other works that will help build the background necessary to fully benefit from the material presented here.

## Conventions for This Book

This book tries to use simple formatting and link directly to source code where necessary. Preformatted blocks of text, like the following,

```
This is a preformatted block of text
```

are used for various content including commands that should be typed into a terminal program prompt, output from terminal programs, data, and other content whose presentation is aided by fixed-width font.

## Communicating with The Project

Join the Rubinius Gitter chatroom [https://gitter.im/rubinius/rubinius](https://gitter.im/rubinius/rubinius) or email <community@rubinius.com>.

## Acknowledgments

Thanks to [Evan Phoenix](https://twitter.com/evanphx) for creating Rubinius and writing a blog post about implementing continuations in the early, stackless virtual machine. When I read that, something clicked and I knew I wanted to work on Rubinius. Evan didn't have any particular qualifications to start a virtual machine project for a programming language rapidly growing in popularity. And he didn't need any. ["Whatever you can do, or dream you can do, begin it. Boldness has genius, power, and magic in it."](http://www.goethesociety.org/pages/quotescom.html)

Thanks to Matz ([Matsumoto Yukihiro](https://en.wikipedia.org/wiki/Yukihiro_Matsumoto)) for creating the Ruby programming language. The best teachers show you what *not* to do, and leave the doing for you to figure out yourself.

Thanks to the many Rubinius contributors for sharing your enthusiasm, time, and patience. Thanks to the many communities working to make tech and industry better for people, and for sharing that widely.

Thanks to Kumiko and Miwa for your understanding, generosity, and patience.

Finally, thanks to my mom for teaching me so much and being my hero for so long.
