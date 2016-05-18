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

We would like to assist building packages and binaries for other Unix/Linux systems. We are removing the Ruby build dependency to make the process of building Rubinius much easier. If you'd like to help build Rubinius binaries or packages, join us in the Rubinius Gitter channel.

## Building from Source Code

Source code tarballs are built for each release. To see which versions are available, visit [http://releases.rubinius.com/index.txt](http://releases.rubinius.com/index.txt).

### Dependencies

Before building Rubinius from source, various dependencies need to be installed, based on your platform.

One of the critical Rubinius dependencies is LLVM version 3.6+. Since LLVM takes a long time to build, we do not support OS versions that do not have modern LLVM binary packages available (e.g. Ubuntu 12.04).

#### Ubuntu/Debian

    $ [sudo] apt-get install -y git clang-3.6 automake flex bison make ruby-dev llvm-dev-3.6 zlib1g-dev libyaml-dev libssl-dev libgdbm-dev libreadline-dev libncurses5-dev libedit-dev

#### Redhat/Fedora

    $ [sudo] yum install -y git clang-3.6 automake flex bison make ruby-devel rubygems llvm-static llvm-devel zlib-devel libyaml-devel openssl-devel gdbm-devel readline-devel ncurses-devel

#### OS X

The easiest way to install dependencies on OS X is to use [Homebrew](http://mxcl.github.com/homebrew/). If it's not already installed, also install the latest version of [Xcode](https://itunes.apple.com/us/app/xcode/id497799835) or the [Command Line Tools](https://github.com/kennethreitz/osx-gcc-installer#readme).

    $ brew install git openssl readline libyaml gdbm llvm38

### Building

    $ [sudo] gem install bundler
    $ curl -OL http://releases.rubinius.com/rubinius-<version>.tar.bz2
    $ tar -xjf rubinius-release-<version>.tar.bz2
    $ cd rubinius-<version>
    $ [sudo] bundle install
    $ ./configure --prefix=/path/to/install/dir/rbx-<version> --llvm-config=$(brew --prefix llvm38)/bin/llvm-config
    $ rake build
    $ [sudo] rake install
