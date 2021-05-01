RUNON - Run your commands in any systems
========================================

(c) 2021 Gilles Grandou <gilles@grandou.net>
licensed under GPL-2.0.

home: https://git.grandou.net/gilles/runon


Quick HOWTO
-----------

    $ grep ^PRETTY_NAME /etc/os-release
    PRETTY_NAME="Debian GNU/Linux 10 (buster)"

    $ runon centos7 grep ^PRETTY_NAME /etc/os-release
    PRETTY_NAME="CentOS Linux 7 (Core)"

    $ runon ubuntu20.04 grep ^PRETTY_NAME /etc/os-release
    PRETTY_NAME="Ubuntu 20.04.2 LTS"

    $ runon debian9 grep ^PRETTY_NAME /etc/os-release
    PRETTY_NAME="Debian GNU/Linux 9 (stretch)"

    $ runon centos7 xclock
    [xclock launched!]


Install
-------

Installation is only tested on Debian 10, even though it should work
straightforward on any equivalent system.

### Docker Install

    sudo apt install docker

Check that your user is part of `docker` group:

    sudo adduser <user> docker

If needed, you need to logout and login again for the new group to
become active.

### manual install

    cd <tools>
    git clone  https://git.grandou.net/gilles/runon

local install, in your `~/local/bin` (or wherever directory which is in your
PATH):

    cd <runon>
    ./install local

or

    ./install local <your_bin_path>

system install, for all users:

    cd <runon>
    sudo ./install system

each user can have its own configuration in `~/.config/runon/runon.conf`
if needed.

### uninstall

simply pass `-u` to install command you have used, eg.:

    ./install local -u
    ./install local -u <dir>
    sudo ./install system -u

### some convenient links

you can create soft links to `runos` to simplify calls:

    runos centos7 -l

now calling `centos7 ...` is equivalent to call `runos centos7 ...`:

    centos7 xclock


Usage
-----

With the default configuration, a seamless environment is set up,
allowing to transparently run commands in various environments, while
keeping:

* user environment (uid, gid, password, home directory, ...)
* X support to run graphical applications

### Basic usage

    runon [options] <osname> <command>

    runon -h

### available options

* `-v` verbose output, this is really usefull when running new
  containers for the first time, as the initial docker build can be
  quite long (several minutes) especially with slow internet link.
  If the command seems to be stalled, don't hesitate to interrupt it
  (with `CTRL-C`) and to restart it with `-v`.

* `-u` forces the container image to be updated, useful if the
  distribution has been updated and you want to use it. Otherwise,
  if a container has been already built, it will be used directly
  without doing any network access.

* `-c <configfile>` uses a custom config file, useful to try new
  distribution without breaking your running config.

### Interactive shell

Just run:

    runon <osname>

while start an insteractive shell in the container system:

    gilles@host:~$ runon centos8
    (centos8) gilles@host:~$ cat /etc/os-release
    NAME="CentOS Linux"
    VERSION="8"
    ID="centos"
    ID_LIKE="rhel fedora"
    VERSION_ID="8"
    PLATFORM_ID="platform:el8"
    PRETTY_NAME="CentOS Linux 8"
    ANSI_COLOR="0;31"
    CPE_NAME="cpe:/o:centos:centos:8"
    HOME_URL="https://centos.org/"
    BUG_REPORT_URL="https://bugs.centos.org/"
    CENTOS_MANTISBT_PROJECT="CentOS-8"
    CENTOS_MANTISBT_PROJECT_VERSION="8"
    (centos8) gilles@host:~$ id
    uid=1000(gilles) gid=1000(gilles) groups=1000(gilles)
    (centos8) gilles@host:~$ sudo id
    [sudo] password for gilles:
    uid=0(root) gid=0(root) groups=0(root)
    (centos8) gilles@host:~$ xclock
    ^C
    (centos8) gilles@host:~$ exit
    exit
    gilles@host:~$


Configuration
-------------

Configuration is done in `runon.conf` file, which describes supported
distribution in .INI format.

### Example config

```
[DEFAULT]
environment =
    HOME
    USER
    DISPLAY
    debian_chroot=${osname}


binds =
    /etc/passwd:ro
    /etc/group:ro
    /etc/shadow:ro
    /tmp/.X11-unix:ro
    /home/${user}

[centos8]
dockerfile =
    FROM centos:8
    RUN yum install dnf-plugins-core -y
    RUN yum config-manager --set-enabled powertools -y
    RUN yum install sudo -y
    RUN echo "Defaults lecture = never" >> /etc/sudoers
    RUN echo "ALL ALL=(ALL) ALL" >> /etc/sudoers
pkginstall = RUN yum install {} -y
packages = ksh csh xterm xorg-x11-apps xkeyboard-config git

[debian9]
dockerfile =
    FROM debian:9
    RUN apt-get update
    RUN apt-get -y install sudo
    RUN echo "Defaults lecture = never" >> /etc/sudoers
pkginstall = RUN apt-get -y install {}
packages = ksh csh xterm x11-apps libgtk-3-0 build-essential git
```

Each section `[osname]` defines a distribution which can be used by runon.
The `[DEFAULT]` section defines default values which is used if not
overriden in individual section.

### Config entries

* `dockerfile` the base content of dockerfile which will be used to
  generate the running environment. There is usually no need to diverge
  from the ones given in example.

* `pkginstall` the dockerfile command used to install a package, likely
  to be standard for all `deb` and `rpm` based distributions. In the
  command `{}` is replaced by the package name.

* `packages` the list of packages to install. Feel free to add the ones
  you need for your commands (likely news system libs, new tools, ...)

* `binds` the list of files and directories from the host system to
  expose in the container system. you might want to add `/opt` or other
  shared directories. See below for a description of `binds` entries

* `environment` the list of environment variables you want to pass or
  set in the container system. See below for a description

Lines starting with `#` or `;` are comments.

Some substitution happens upon reading the configuration:

* `${user}` the current username
* `${osname}` the executed distribution.

### Binds

Each `binds` line can have one of the following formats:

    <hostpath>
    <hostpath>:<mode>
    <hostpath>:<containerpath>
    <hostpath>:<containerpath>:<mode>

with:

* `<hostpath>` is the filename or the dirname of the path you want
  to expose
* `<containerpath>` is the pathname inside the container, by default
  it's the same path.
* `<mode>` can be `rw`, read-write (by default), or `ro`, read-only.

### Environment

Each `environment` line define a Environment Variable which is set
in the container upon execution.

Each line can have one of the following formats:

    <envvar>
    <envvar>=<value>

If no value is given, the host value is passed into the container.

