# Releases

As described in the [versioning scheme](/part_i_versioning.html) section, a Rubinius release is a pair (EPOCH.SEQ, SHA) representing a function from version numbers to git commit SHAs.

A _release_ is implemented as a git tag on the master branch, and from this we can automatically derive the version number, the date of the release, and the commit SHA.

A _release artifact_ is a source tarball, Docker image, or binary package for a particular platform.

The Rubinius release process is fully automated and initiated by pushing a git tag of the form "vX.Y". Any Rubinius contributor is able to create a Rubinius release. The command to release is quite simple.

```
% scripts/tag.sh tag
Fetching origin
From https://github.com/rubinius/rubinius
Tagging v3.34 as "Version 3.34 (2016-06-04)"
% git push origin master --tags # push tag to repository which kicks off release mechanism
```

When the git tag is pushed to GitHub, an automated process builds various release artifacts and updates [the Rubinius website](http://rubinius.com).

The Rubinius release process makes trade-offs in favor of distributed, collaborative work without forcing people to synchronize and agree in advance. If a bad commit is tagged, any contributor can remove the git tag and push a new one. This provides resilience in the process and facilitates constant improvement.

## Updating Standard Library Gems

The `Standard Library` may evolve at a different rate than the core system. Rubinius packages its standard library as gems and uses a `gems_list.txt` file along with `bundler` to manage the gem release and gem versioning process. When a new version of a standard library gem has been released (which means it was pushed to rubygems.org) then there is a short process to follow to update Rubinius.

To start, one must use a recently built Rubinius that is in the `$PATH` (perhaps via `chruby` or similar manager). The following operations must be performed on a git repository that does *not* contain the currently active/running Rubinius in $PATH.

```
% ./configure && ./scripts/gems.sh update_list
Checking clang: found
Checking clang++: found

Detected old configuration settings, forcing a clean build
  Checking for 'llvm-config': found! (version 3.8.0 - api: 308)
... eliding lines
Fetching gem metadata from https://rubygems.org/
Fetching version metadata from https://rubygems.org/
Resolving dependencies........
... eliding lines
Bundle updated!
... eliding lines
%
```

The script at `./scripts/gems.sh` is the canonical source, but effectively it is calling `bundle update` and then recording the names and versions of all gems in the bundle to store away in the `gems_list.txt` file.

## Updating Preinstalled Gems

Certain gems are preinstalled with every release of Rubinius. They are not part of the Standard Library but the Rubinius build and release process expects these gems to exist.

```ruby
gems = ["bundler-1.11.2.gem",
        "minitest-5.11.1.gem",
        "racc-1.4.14.gem",
        "rake-12.1.0.gem",
        "rdoc-4.3.0.gem",
        "rb-readline-0.5.4.gem",
        "rubysl-readline-2.0.2.gem"
       ]
```

For example, `rake` version 12.1.0 is installed as part of the process. However, as of this writing the latest version of `rake` is 12.3.0. To update to this latest version, modify `build/scripts/preinstall_gems.rb` and edit the version number in that file. See below:

```ruby
gems = ["bundler-1.11.2.gem",
        "minitest-5.11.1.gem",
        "racc-1.4.14.gem",
        "rake-12.3.0.gem", # <--- update version number to 12.3.0 from 12.1.0
        "rdoc-4.3.0.gem",
        "rb-readline-0.5.4.gem",
        "rubysl-readline-2.0.2.gem"
       ]
```

Second, the `gems_list.txt` file should be manually edited to include this new version number.

After making these changes, be certain to rebuild Rubinius locally to confirm it builds correctly. In some cases it might be necessary to clean all untracked foreign files from the repository for a clean build. Run the command `git clean -fdx` to remove files and directories that are not part of the repository. Then execute the build and confirm it succeeds.
