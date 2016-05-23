# Releases

As described in the [versioning scheme](/part_i/versioning.html) section, a Rubinius release is a pair (EPOCH.SEQ, SHA) representing a function from version numbers to git commit SHAs.

A _release_ is implemented as a git tag on the master branch, and from this we can automatically derive the version number, the date of the release, and the commit SHA.

A _release artifact_ is a source tarball, Docker image, or binary package for a particular platform.

The Rubinius release process is fully automated and initiated by pushing a git tag of the form “vX.Y”. Any Rubinius contributor is able to create a Rubinius release.

When the git tag is pushed to GitHub, an automated process builds various release artifacts and updates [the Rubinius website](http://rubinius.com).

The Rubinius release process makes trade-offs in favor of distributed, collaborative work without forcing people to synchronize and agree in advance. If a bad commit is tagged, any contributor can remove the git tag and push a new one. This provides resilience in the process and facilitates constant improvement.
