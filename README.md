:rotating_light: The project has moved to a self-hosted git instance!<br/>
:rotating_light: Please use the new URL for an up-to-date version: https://code.apps.glenux.net/glenux-contrib/vagrant-lxc

🟢 We plan to support and maintain vagrant-lxc, as well as clean it up.<br/>
🟢 Please feel free to contribute Issues and pull requests.<br/>
🟢 P.S: Thanks Fabio Rehm for the amazing initial project.

# vagrant-lxc

[![Build Status](https://travis-ci.org/fgrehm/vagrant-lxc.png?branch=master)](https://travis-ci.org/fgrehm/vagrant-lxc) [![Gem Version](https://badge.fury.io/rb/vagrant-lxc.png)](http://badge.fury.io/rb/vagrant-lxc) [![Code Climate](https://codeclimate.com/github/fgrehm/vagrant-lxc.png)](https://codeclimate.com/github/fgrehm/vagrant-lxc) [![Coverage Status](https://coveralls.io/repos/fgrehm/vagrant-lxc/badge.png?branch=master)](https://coveralls.io/r/fgrehm/vagrant-lxc) [![Gitter chat](https://badges.gitter.im/fgrehm/vagrant-lxc.png)](https://gitter.im/fgrehm/vagrant-lxc)

[LXC](http://lxc.sourceforge.net/) provider for [Vagrant](http://www.vagrantup.com/) 1.9+

This is a Vagrant plugin that allows it to control and provision Linux Containers
as an alternative to the built in VirtualBox provider for Linux hosts. Check out
[this blog post](http://fabiorehm.com/blog/2013/04/28/lxc-provider-for-vagrant/)
to see it in action.

## Features

* Provides the same workflow as the Vagrant VirtualBox provider
* Port forwarding via [`redir`](https://github.com/troglobit/redir)
* Private networking via [`pipework`](https://github.com/jpetazzo/pipework)

## Requirements

* [Vagrant 1.9+](http://www.vagrantup.com/downloads.html)
* lxc >=2.1
* `redir` (if you are planning to use port forwarding)
* `brctl` (if you are planning to use private networks, on Ubuntu this means `apt-get install bridge-utils`)

The plugin is known to work better and pretty much out of the box on Ubuntu 14.04+
hosts and installing the dependencies on it basically means a
`apt-get install lxc lxc-templates cgroup-lite redir`. For setting up other
types of hosts please have a look at the [Wiki](https://github.com/fgrehm/vagrant-lxc/wiki).

If you are on a Mac or Windows machine, you might want to have a look at [this](http://the.taoofmac.com/space/HOWTO/Vagrant)
blog post for some ideas on how to set things up or check out [this other repo](https://github.com/fgrehm/vagrant-lxc-vbox-hosts)
for a set of Vagrant VirtualBox machines ready for vagrant-lxc usage.


## Installation

```
vagrant plugin install vagrant-lxc
```


## Quick start

```
vagrant init fgrehm/precise64-lxc
vagrant up --provider=lxc
```

_More information about skipping the `--provider` argument can be found at the
"DEFAULT PROVIDER" section of [Vagrant docs](https://docs.vagrantup.com/v2/providers/basic_usage.html)_

## Base boxes

Base boxes provided on Atlas haven't been refreshed for a good while and shouldn't be relied on.
Your best best is to build your boxes yourself. Some scripts to build your own are available at
[hsoft/vagrant-lxc-base-boxes](https://github.com/hsoft/vagrant-lxc-base-boxes).

If you want to build your own boxes, please have a look at [`BOXES.md`](https://github.com/fgrehm/vagrant-lxc/tree/master/BOXES.md)
for more information.

## Advanced configuration

You can modify container configurations from within your Vagrantfile using the
[provider block](http://docs.vagrantup.com/v2/providers/configuration.html):

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "fgrehm/trusty64-lxc"
  config.vm.provider :lxc do |lxc|
    # Same effect as 'customize ["modifyvm", :id, "--memory", "1024"]' for VirtualBox
    lxc.customize 'cgroup.memory.limit_in_bytes', '1024M'
  end
end
```

vagrant-lxc will then write out `lxc.cgroup.memory.limit_in_bytes='1024M'` to the
container config file (usually kept under `/var/lib/lxc/<container>/config`)
prior to starting it.

For other configuration options, please check the [lxc.conf manpages](http://manpages.ubuntu.com/manpages/precise/man5/lxc.conf.5.html).

### Private Networks

Starting with vagrant-lxc 1.1.0, there is some rudimentary support for configuring
[Private Networks](https://docs.vagrantup.com/v2/networking/private_network.html)
by leveraging the [pipework](https://github.com/jpetazzo/pipework) project.

On its current state, there is a requirement for setting the bridge name that
will be created and will allow your machine to comunicate with the container

For example:

```ruby
Vagrant.configure("2") do |config|
  config.vm.network "private_network", ip: "192.168.2.100", lxc__bridge_name: 'vlxcbr1'
end
```

Will create a new `veth` device for the container and will set up (or reuse)
a `vlxcbr1` bridge between your machine and the `veth` device. Once the last
vagrant-lxc container attached to the bridge gets `vagrant halt`ed, the plugin
will delete the bridge.

### Container naming

By default vagrant-lxc will attempt to generate a unique container name
for you. However, if the container name is important to you, you may use the
`container_name` attribute to set it explicitly from the `provider` block:

```ruby
Vagrant.configure("2") do |config|
  config.vm.define "db" do |node|
    node.vm.provider :lxc do |lxc|
      lxc.container_name = :machine # Sets the container name to 'db'
      lxc.container_name = 'mysql'  # Sets the container name to 'mysql'
    end
  end
end
```

_Please note that there is a 64 chars limit and the container name will be
trimmed down to that to ensure we can always bring the container up.

### Backingstore options

Support for setting `lxc-create`'s backingstore option (`-B` and related) can be
specified from the provider block and it defaults to `best`, to change it:

```ruby
Vagrant.configure("2") do |config|
  config.vm.provider :lxc do |lxc|
    lxc.backingstore = 'lvm' # or 'btrfs', 'overlayfs', ...
    # lvm specific options
    lxc.backingstore_option '--vgname', 'schroots'
    lxc.backingstore_option '--fssize', '5G'
    lxc.backingstore_option '--fstype', 'xfs'
  end
end
```

## Unprivileged containers support

Since v1.4.0, `vagrant-lxc` gained support for unprivileged containers. For now, since it's a new
feature, privileged containers are still the default, but you can have your `Vagrantfile` use
unprivileged containers with the `privileged` flag (which defaults to `true`). Example:

```ruby
Vagrant.configure("2") do |config|
  config.vm.provider :lxc do |lxc|
    lxc.privileged = false
  end
end
```

For unprivileged containers to work with `vagrant-lxc`, you need a properly configured system. On
some distros, it can be somewhat of a challenge. Your journey to configuring your system can start
with [Stéphane Graber's blog post about it](https://stgraber.org/2014/01/17/lxc-1-0-unprivileged-containers/).

## Avoiding `sudo` passwords

If you're not using unprivileged containers, this plugin requires **a lot** of `sudo`ing To work
around that, you can use the `vagrant lxc sudoers` command which will create a file under
`/etc/sudoers.d/vagrant-lxc` whitelisting all commands required by `vagrant-lxc` to run.

If you are interested on what will be generated by that command, please check
[this code](lib/vagrant-lxc/command/sudoers.rb).


## More information

Please refer the [wiki](https://github.com/fgrehm/vagrant-lxc/wiki).


## Problems / ideas?

Please review the [Troubleshooting](https://github.com/fgrehm/vagrant-lxc/wiki/Troubleshooting)
wiki page + [known bugs](https://github.com/fgrehm/vagrant-lxc/issues?labels=bug&page=1&state=open)
list if you have a problem and feel free to use the [issue tracker](https://github.com/fgrehm/vagrant-lxc/issues)
propose new functionality and / or report bugs.


## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
