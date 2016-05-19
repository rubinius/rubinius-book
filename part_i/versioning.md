# Versioning

There are a lot of versioning schemes in software, from lax to strict to weird and everything in between. Versioning schemes are the source of endless debates and plenty of disappointment and unhappiness all around. That’s unfortunate because a versioning scheme is about getting new stuff. Who doesn’t want new stuff?

A versioning scheme is part communication and part contract. The communication part is a signal from developers that an update is available. The contract part is an agreement about the impact of the update.

The versioning scheme exists in the context of two competing concerns: 1. the developer wants to deliver features and improvements; and 2. the user wants the highest stability for the least amount of effort on their part.

The conflict between these two competing concerns is complicated. Every user’s needs are unique (even allowing for some groups of similar users). Trying to devise a scheme that meets all of those needs at once is impossible. Especially as the scope of the software covered by a single version number increases. Consequently, there’s a tension between batching things up in big enough chunks to accommodate the users who update slowly and providing new features quickly to those that update often.

To complicate this even further, the software development industry still suffers from the legacy of putting bits on physical media, putting those into boxes, into trucks, and onto store shelves. We are slowly moving away from this legacy model and to one where a mostly invisible stream of improvements find their way automatically to our devices and applications. This "evergreen" model of software updates is most visible (or invisible) in web browsers, but is also becoming common on our smart phones, too.

## Rubinius Versioning

The Rubinius versioning scheme associates a version number, in the form of EPOCH.SEQ, with a particular git commit SHA via a git tag.

The first part of the version number, the EPOCH, signifies “a period of time marked by notable events or particular characteristics”. The SEQ is a monotonically increasing number that has no other meaning than to signal that newer code is available.

There are two distinct things here. The _version number_ is the EPOCH.SEQ. For example, version 3.42. The git commit SHA labels a unique set of bits that represent one instant in the history of the software source code. The combination of these two, in the form of a pair (EPOCH.SEQ, SHA), label a [Rubinius release](/part_i/releases.html).

As mentioned above, a version number is part communication and part contract.

_Communication:_ If you are using Rubinius version A.N and there exists a version A.M, where M > N, there is new code available and you should upgrade.

_Contract:_ If you are using Rubinius version A.N, version A.N+1 will do one or more of the following: 1. add new features; 2. remove a previously deprecated feature; or 3. add a deprecation warning. In other words, if you have no deprecation warnings with A.N, you can update to A.N+1 and expect no breaking changes.

This Rubinius versioning scheme helps us deliver features faster. And it explicitly decouples Rubinius changes from your decision about when to upgrade. We provide a clear signal and a clear expectation about the impact of a new version. If you decide to update every 15 or 20 versions, you can do so in whatever way works best for you. You can jump ahead to the newest version and if that doesn’t work, bisect version numbers just like you might bisect git commits to find an issue.
