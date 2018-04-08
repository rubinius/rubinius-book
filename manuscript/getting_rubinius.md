# Getting Rubinius

Rubinius can be installed on most Unix/Linux operating systems. Rubinius does not run on Windows yet.

## Installing Packages

### Docker

Rubinius provides Docker images based on Ubuntu 14.04. The images are automatically built on Docker Hub and available at the Rubinius Docker organization. To use the Rubinius Docker image, follow these steps:

      your-shell$ docker pull rubinius/docker
      your-shell$ docker run -it rubinius/docker bash
    docker-shell# ruby -v

### OS X Homebrew

Rubinius provides Homebrew binaries that should be compatible with 10.8 (Mountain Lion) and newer OS X releases. To install Rubinius on Homebrew, follow these steps:

    $ brew update
    $ brew tap rubinius/apps
    $ brew install rubinius

### Ubuntu

We build binaries for Ubuntu for 14.04. These binaries are built so that they can be used on Travis with RVM. They can also be manually expanded for use with chruby.

To see all the binaries that are available, visit [http://binaries.rubinius.com/index.txt](http://binaries.rubinius.com/index.txt).

### Heroku

We build binaries for Heroku. Specify the latest Rubinius version in your applications Gemfile and when you push to Heroku, their build system will use the binary we built for that version. An example for your Gemfile follows:

    ruby "2.2.2", :engine => "rbx", :engine_version => "3.31"


### Other Unix/Linux


We highly recommend using [ruby-install](https://github.com/postmodern/ruby-install) to build Rubinius instead of rbenv or RVM. ruby-install does a good job of making sure the pre-reqs are met, including getting a build ruby.

We intend to eventually provide packages and binaries for other Unix/Linux systems. If you are interested in helping us get a Concourse CI build pipeline going, or in building binaries for a particular OS and sharing them, join us in the Rubinius Gitter channel.


## Building from Source Code

Source code tarballs are built for each release. To see which versions are available, visit [http://releases.rubinius.com/index.txt](http://releases.rubinius.com/index.txt).

### Dependencies

Rather than build Rubinius from source, we recommend using [ruby-install](https://github.com/postmodern/ruby-install). ruby-install does a good job of making sure the prereqs are met, including getting a build ruby. Eventually, the ruby build dependency will go away, and building from source will be as easy as `./build.sh`. But until then, we recommend giving ruby-install a go.

Before building Rubinius from source, various dependencies need to be installed, based on your platform.

One of the critical Rubinius dependencies is LLVM version 3.6+. Since LLVM takes a long time to build, we do not support OS versions that do not have modern LLVM binary packages available (e.g. Ubuntu 12.04).


#### Ubuntu/Debian

    $ [sudo] apt-get install -y git clang-3.6 automake flex bison make ruby-dev llvm-dev-3.6 zlib1g-dev libyaml-dev libssl-dev libgdbm-dev libreadline-dev libncurses5-dev libedit-dev

Use the following configure command in the build instructions below:

    $ ./configure --prefix=/path/to/install/dir/rbx-<version> --llvm-config=llvm-config-3.6

**NOTE: If you are using LLVM 4.0, you _must_ link only with libc++. Use the following configure command in the build instructions below:**

    $ CXXFLAGS='-nostdinc++ -I/usr/include/c++/v1' LDFLAGS='-stdlib=libc++ -lc++' ./configure --prefix=/path/to/install/dir/rbx-<version> --llvm-config=llvm-config-4.0 --cc=clang-4.0 --cxx=clang++-4.0

#### Redhat/Fedora

    $ [sudo] yum install -y git clang-3.6 automake flex bison make ruby-devel rubygems llvm-static llvm-devel zlib-devel libyaml-devel openssl-devel gdbm-devel readline-devel ncurses-devel

Use the following platform-specific configure command in the build instructions below:

    $ ./configure --prefix=/path/to/install/dir/rbx-<version>

#### OS X

The easiest way to install dependencies on OS X is to use [Homebrew](http://mxcl.github.com/homebrew/). If it's not already installed, also install the latest version of [Xcode](https://itunes.apple.com/us/app/xcode/id497799835) or the [Command Line Tools](https://github.com/kennethreitz/osx-gcc-installer#readme).

    $ brew install git openssl readline libyaml gdbm llvm38

Use the following platform-specific configure command in the build instructions below:

    $ ./configure --prefix=/path/to/install/dir/rbx-<version> --llvm-config=$(brew --prefix llvm38)/bin/llvm-config-3.8

### Building

In the interest of making it easier to build Rubinius, we are in the process of replacing the conventional (ruby-dependent) build process with the one located at `./build.sh`. It is very much a work in progress, but once complete it will handle all platform dependencies, etc without requiring an initial MRI ruby install.

For now, here's the conventional (ruby-dependent) way of building:

    $ [sudo] gem install bundler
    $ curl -OL http://releases.rubinius.com/rubinius-<version>.tar.bz2
    $ tar -xjf rubinius-release-<version>.tar.bz2
    $ cd rubinius-<version>
    $ [sudo] bundle install
    $ <platform-specific configure command>
    $ rake build
    $ [sudo] rake install
