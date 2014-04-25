 Naming them
  Deprecating them
  Good API examples vs Bad?
  Semantic Versioning quick guide in the context of modules.  (Fix a bug - 1.0.x.  Add parameters - 1.x.0.  Change existing parameters.  x.0.0)

#Classes, parameters, and APIs

As mentioned in the [Beginner's Guide to
Modules](http://docs.puppetlabs.com/guides/module_guides/bgtm.html),
*classes* are used to aggregate and organize *parameters*, the
publically-consumable *API*-like interface of your module.  This first
section of the Advanced Guide to Modules will walk through the best
practices for developing a basic module: from determining how many and
which classes to use to figuring out what your parameters will do and
which should be public vs. private. To help illustrate these
practices, we'll build the framework of a basic ssh module.

##Getting Started

###Intro to SSH

SSH is software designed to secure connections to remote servers.  It
can be used to provide a secure shell, as well as to transfer files
and tunnel TCP traffic securely through machines.  For more
information please visit
[Wikipedia](http://en.wikipedia.org/wiki/Secure_Shell).

###Classes

A module is, at its heart, a place to create a number of classes that
belong together.  We create classes to separate out the logic of a
module into a number of smaller, self-contained tasks/functions that
are organized around a single purpose.  The most common pattern is

* class
* class::install
* class::config
* class::service
* class::other

Visualize this pattern as a division between private and public
interfaces: You have a single public class (i.e. `class`) that allows
you to centralize the management of the parameters and logic required
for the configuration of the software the module manages, then smaller
private subclasses that exist purely to batch together similar
functionality and make it easier for developers to isolate the
internal implementations of module functionality (such as configuring
all the configuration files).  It's almost the same as developing
functions in real languages, they exist to isolate functionality and
enable safe refactoring without having to change other pieces of the
system.

Where possible, names of classes should be self-evident and organized
underneath the public class.  For instance,

* class::server
* class::server::private_class

##Parameters

One of the hardest parts of building a module is designing the Public
API, the parameters for all the classes.  A module with too few
parameters is inflexible, whilst a module with too many parameters
loses focus and thus efficacy because it has no opinions.  Ideally
your module would be opinionated without losing all flexibility,
keeping in mind the scope of your module.  This might mean creating
parameters for the most common options you wish to configure, as well
as with escape-hatch parameters that allow a user to pass in custom
files or hashes.

In this section, we will cover Puppet Labs' best practices for
[naming](#naming) parameters, deciding how [many](#amount) parameters
to create, and appropriately
[separating](#separating-logic-from-configuration) module parameters.

###Naming

The first rule of naming parameters is simple: "Make them obvious."
Parameters with obvious names include

* package_ensure
* service_enable
* service_manage

All of the above parameters rely on combining the resource type with
the property to make obvious, self-documenting names.  For
application-specific parameters, we recommend using the terms
contained within the product, such as

* keys_file
* panic
* restrict
* servers

The above parameters were taken from the [ntp
module](http://forge.puppetlabs.com/puppetlabs/ntp) and are mostly
examples of single-word parameters.  We could have used names like
`servers_list`, `restrict_array`, or`panic_boolean`, but we must
assume that the user generally understands the configuration file
options commonly used in NTP and knows what type of data they require
(a list of servers vs a true/false value, for example).

It's a mistake to make parameters with names that don't reflect the
knowledge of experienced users of the software you're managing in
order to increase the ease of use for inexperienced users.  It's
easier for them to learn the technology than for experienced users to
learn your terminology. Naming a parameter `local_clock_boolean`
instead of `udlc` would confuse an experienced NTP user and force them
to check the README or code to ensure they understood what the
parameter did.

In order to assist users reading the code of your module, you should
separate out the parameters that control configuration options from
those parameters that control the logical flow of the module.  This
means you may end up with a class that looks like:

```
  class software(
    # Logic
    $service_manage,
    $epel,
    $dev_enable,
    # Configuration
    $servers_list,
    $blacklist,
    $whitelist,
  ) {}
```

###Scoping (size/responsibility/different word)

Scoping the purpose and size of your module is critical.  In order to
build a module to manage SSH we must consider what functionality falls
under “managing SSH”.  It’s relatively easy to create a list of things
that would fall under this category:

* Managing the ssh packages.
* Managing the ssh service.
* Global ssh client configuration.
* Global ssh server configuration.
* Individual user ssh configuration.

Scoping gets harder when you consider related functionality like:

* Managing ssh host keys.
* Managing individual user ssh keys.

However, both of those tasks can be done independently of managing ssh
as a daemon and therefore would be better off in a separate module,
allowing them to be used by someone who already has an existing method
by which they manage the ssh daemon.

For the purposes of this guide we’ll be working to manage the five
items listed above.

###Managing the ssh packages.

We'll start our ssh module using the puppet module tool (PMT):

```
  $ puppet module generate puppetlabs-ssh
  Notice: Generating module at /Users/apenney/modules/puppetlabs-ssh
  puppetlabs-ssh
  puppetlabs-ssh/Modulefile
  puppetlabs-ssh/README
  puppetlabs-ssh/manifests
  puppetlabs-ssh/manifests/init.pp
  puppetlabs-ssh/spec
  puppetlabs-ssh/spec/spec_helper.rb
  puppetlabs-ssh/tests
  puppetlabs-ssh/tests/init.pp
```

Using the PMT is the suggested way of creating modules, as it provides
a quick and easy way of obtaining all of the basic module files we'll
need.

We’ll start by creating some skeleton classes for our module so we
have a place to manage packages from.  The below tree output shows the
tree of classes for both client and manifests server.

```
$ tree
.
├── client
│   └── install.pp
├── client.pp
├── init.pp
├── params.pp
├── server
│   ├── install.pp
└── server.pp
```

This includes the main ssh::server and ssh::client classes as well as
the specific install classes for both pieces.  We split out the server
and client right at the start so that they can be individually
managed.  The main client and server classes look like:

```
class ssh::client (
  $package_ensure = $ssh::params::client_package_ensure,
  $package_name   = $ssh::params::client_package_name,
) inherits ssh::params {

  contain ssh::client::install

}
```

```puppet
class ssh::server(
  $package_ensure            = $ssh::params::server_package_ensure,
  $package_name              = $ssh::params::server_package_name,
) inherits ssh::params {

  contain ssh::server::install

}
```

As discussed in the parameters section we’ve used the pattern of
“resource_property” to create these parameter names.

If `contain` is new to you then a brief description is that it
replaces the anchor{} resource of the past and allows you to properly
contain classes within the scope of other classes so that they don’t
drift.  Further information can be found at [Language
Containment](http://docs.puppetlabs.com/puppet/latest/reference/lang_containment.html)

This leaves us with ::install classes that look like:

```
class ssh::client::install inherits ssh::client {

  if $caller_module_name != $module_name {
    fail("Use of private class ${name} by ${caller_module_name}")
  }

  package { 'ssh':
    ensure => $package_ensure,
    name   => $package_name,
  }

}

class ssh::server::install inherits ssh::server {

  if $caller_module_name != $module_name {
    fail("Use of private class ${name} by ${caller_module_name}")
  }

  package { 'sshd':
    ensure => $package_ensure,
    name   => $package_name,
  }

}
```

While most of these classes are unsurprising we do two things that are
less common in existing modules.  First, these classes inherit the
main class.  This brings the parameters from ssh::server and
ssh::client in scope to these private classes, allowing us to simply
reference $package_ensure rather than having to pass the parameters
into these classes.  Secondly, we explicitly mark these classes as
being private with:

```
  if $caller_module_name != $module_name {
    fail("Use of private class ${name} by ${caller_module_name}")
  }
```

Which causes compilation of the Puppet catalog to fail if the class
was included anywhere by in the scope of a class within the module.
If you were to add ssh::server::install to site.pp or to the console
of Puppet Enterprise it would immediately fail.

###Managing the ssh service

We’ll extend our module by adding a new class, ssh::server::service.
Earlier, when we made the decision to split the classes into ::server
and ::client, it was influenced by the realization that there’s no
such thing as a client service, so we would have needed to introduce
logic to protect against managing the service if you were only trying
to manage the client.

```
class ssh::server::service inherits ssh::server {

  if $caller_module_name != $module_name {
    fail("Use of private class ${name} by ${caller_module_name}")
  }

  service { 'sshd':
    ensure => $service_ensure,
    name   => $service_name,
  }

}
```

We then need to modify our main ssh::server class to include this:

```
class ssh::server(
  $package_ensure            = $ssh::params::server_package_ensure,
  $package_name              = $ssh::params::server_package_name,
  $service_ensure            = $ssh::params::server_service_ensure,
  $service_name              = $ssh::params::server_service_name,
) inherits ssh::params {

  # We declare the classes before containing them.
  class { 'ssh::server::install': } ->
  class { 'ssh::server::service': }

  contain ssh::server::install
  contain ssh::server::service
}
```

The biggest change here is the introduction of the `->` chaining
syntax.  This sets up ordering between the classes, as documented at
[docs.puppetlabs.com](http://docs.puppetlabs.com/learning/ordering.html),
ensuring that the ssh::server::install class runs before the
ssh::server::service class.  Without this we run the risk of service
attempting to run before the ssh packages are installed.

We prefer to create dependencies at the class level rather than at the
resource level in order to free ourselves up to internally refactor
the classes without having to constantly remember to update
dependencies in numerous resources.  This pattern also naturally lends
itself to handling external dependencies, such as:

Class['apt'] -> Class['ssh::server']

This would allow you to ensure the contents of the apt module ran
before any of the ssh::server subclasses attempted to do anything with
packages.

There will be times when it makes sense to break these rules, but it
should be infrequent.  By moving your composition of dependencies to
the class level you shield yourself from constantly updating
dependencies whenever the internals of a class changes.  It is better
to split a class into two if you need to depend on only half of it
than it is to fall into the antipattern of requiring resources.

###Global ssh client configuration

Now we wish to address the global ssh client configuration, which is
stored in a file called ssh_config.  We first create a
ssh::client::config class:

```
class ssh::client::config inherits ssh::client {

  if $caller_module_name != $module_name {
    fail("Use of private class ${name} by ${caller_module_name}")
  }

  file { 'ssh_config':
    ensure => $config_ensure,
    path   => $config_path,
    content => template('ssh/ssh_config.erb'),                                  
  }

}
```

We then modify the main ssh::client class to include these new
parameters and this class:

```
class ssh::client (
  $config_ensure  = $ssh::params::client_config_ensure,
  $config_path    = $ssh::params::client_config_path,
  $package_ensure = $ssh::params::client_package_ensure,
  $package_name   = $ssh::params::client_package_name,
) inherits ssh::params {

  # We declare the classes before containing them.
  class { 'ssh::client::install': } ->
  class { 'ssh::client::config': }

  contain ssh::client::install
  contain ssh::client::config
}
```

With the framework for configuring the SSH client in place we need to
leverage our domain knowledge of SSH to pick out things that users are
likely to configure.  The ssh_config file is constructed of a Host
entry, and then a number of indented parameters below that host entry.

There’s a number of choices that can be made here as to how flexible
to be with this module.  We may wish to allow you to set any number of
hosts and their parameters, and we’d need to use defines and possibly
puppetlabs-concat to do this.

For the purposes of this module we’ll exclusively manage ‘Host *’,
parameters that apply to all hosts.  We’ll keep things simple by
managing the following things:

* ForwardAgent
* ForwardX11
* PasswordAuthentication
* Port
* Protocol

This means our template will look like:

```
##
## Managed by Puppet
##

Host *
  ForwardAgent <%= forward_agent %>
  ForwardX11 <%= forward_x11 %>
  PasswordAuthentication <%= password_authentication %>
  Port <%= port %>
  Protocol <%= protocol %>
```

We add these parameters to ssh::client, which becomes:

```
  class ssh::client (
    $config_ensure           = $ssh::params::client_config_ensure,
    $config_path             = $ssh::params::client_config_path,
    $package_ensure          = $ssh::params::client_package_ensure,
    $package_name            = $ssh::params::client_package_name,
    # Configuration
    $forward_agent           = $ssh::params::client_forward_agent,
    $forward_x11             = $ssh::params::client_forward_x11,
    $password_authentication = $ssh::params::client_password_authentication,
    $port                    = $ssh::params::client_port,
    $protocol                = $ssh::params::client_protocol,
) inherits ssh::params {

  # We declare the classes before containing them.
  class { 'ssh::client::install': } ->
  class { 'ssh::client::config': }

  contain ssh::client::install
  contain ssh::client::config
}    
```

This is a simplified selection, and it raises the question of why we
didn’t choose to manage all of the possible parameters for
sshd_config.  There’s several answers here, but the most important is
that a module with tens, or hundreds, of parameters is simply too
unwieldy to use, or develop, and it becomes extremely difficult to
maintain with time.  There’s several causes of too many parameters,
but two that you are most likely to face are too broad of a module
scope, or reliance on individual parameters rather than more
sophisticated data handling.

The first case is simple; in some cases the module scope is simply too
broad.  You’re trying to manage too many things and you’d be better
off breaking the module into several.

The second case is more complex to deal with but it comes down to data
structures.  In the case of ssh_config we could have created a
parameter per piece of configuration you can pass to a host, but that
gets complex and requires you to keep updating the module as ssh
changes.

Given a configuration file of:

Host a
  Parameter 1

Host b
  Parameter 2

You could create a `configuration_data` parameter in ssh::client and
pass it:

[ {‘a’ => {‘Parameter’ => ‘1’}}, {‘b’ => {‘Parameter’ => ‘2’}} ]

This is an array of hashes, each of which contains a key that points
to another sub-hash of parameters.  It’s complex if you’re not used to
dealing with nested data structures, but it’s arbitrarily expandable
to any number of hosts, and parameters, and is contained within a
single parameter meaning that any possible ssh_configuration data is
expressible.

To see this in action we can modify our ssh::client work as follows:

```
  class ssh::client (
    $config_ensure           = $ssh::params::client_config_ensure,
    $config_path             = $ssh::params::client_config_path,
    $package_ensure          = $ssh::params::client_package_ensure,
    $package_name            = $ssh::params::client_package_name,
    # Configuration
    $configuration_data      = $ssh::params::client_configuration_data,
) inherits ssh::params {

  # We declare the classes before containing them.
  class { 'ssh::client::install': } ->
  class { 'ssh::client::config': }

  contain ssh::client::install
  contain ssh::client::config
  }       
```

We need to modify the template next:

```
##
## Managed by Puppet
##

<% @configuration_data.sort.each do |hash| -%>
<%   hash.each do |host, parameters| -%>
Host <%= host %>
<%     if parameters.is_a?(Hash) -%>
<%       parameters.sort.map do |title, value| -%>
  <%= title %> <%= value %>
<%       end -%>
<%     end -%>
<%   end -%>
<% end -%>
```

While this is a little more complex, it basically boils down to
iterating over each hash within the array, then printing the key out
as the name of the Host, then a further iteration over all the
parameters contained in the subhash.  The end result looks like:

```
##
## Managed by Puppet
##

Host *
  Port 22
```

###Global ssh server configuration

We now extend the module in exactly the same way to cover
ssh::server::config.  This is slightly less complex in that we don’t
need to use nested hashes, just a single hash with entries for each
parameter and their value.  We start by making the ssh::server::config
class and modifying the ssh::server class to include it.

```
class ssh::server::config inherits ssh::server {

  if $caller_module_name != $module_name {
    fail("Use of private class ${name} by ${caller_module_name}")
  }

  file { 'sshd_config':
    ensure  => $config_ensure,
    path    => $config_path,
    owner   => 'root',
    group   => 'root',
    mode    => '0544',
    content => template('ssh/sshd_config.erb'),
  }

}

class ssh::server(
  $config_ensure             = $ssh::params::server_config_ensure,
  $config_path               = $ssh::params::server_config_path,
  $package_ensure            = $ssh::params::server_package_ensure,
  $package_name              = $ssh::params::server_package_name,
  $service_ensure            = $ssh::params::server_service_ensure,
  $service_name              = $ssh::params::server_service_name,
  # Configuration parameters
  $configuration_data        = $ssh::params::server_configuration_data,
  $os_configuration_data     = $ssh::params::server_os_configuration_data,
) inherits ssh::params {

  # We declare the classes before containing them.
  class { 'ssh::server::install': } ->
  class { 'ssh::server::config': } ~>
  class { 'ssh::server::service': }

  contain ssh::server::install
  contain ssh::server::config
  contain ssh::server::service

}
```

We’ve introduced two new parameters in the above class,
`configuration_data` and `os_configuration_data`.  This makes it
easier to provide data that differs per operating system in params.pp
while letting you override the other sshd settings.

Next we update params.pp to include all the new data.  This file has
grown at this point to handle Redhat and Debian, as well as provide
sensible defaults:

```
class ssh::params {

  # Server parameters
  $server_config_ensure           = 'present'
  $server_package_ensure          = 'present'
  $server_service_ensure          = 'running'
  $server_configuration_data      = {
    'Port' => '22',
    'Protocol' => '2',
    'UsePrivilegeSeparation' => 'yes',
    'KeyRegenerationInterval' => '3600',
    'ServerKeyBits' => '768',
    'SyslogFacility' => 'AUTH',
    'LogLevel' => 'INFO',
    'LoginGraceTime' => '120',
    'PermitRootLogin' => 'yes',
    'StrictModes' => 'yes',
    'RSAAuthentication' => 'yes',
    'PubkeyAuthentication' => 'yes',
    'IgnoreRhosts' => 'yes',
    'RhostsRSAAuthentication' => 'no',
    'HostbasedAuthentication' => 'no',
    'PermitEmptyPasswords' => 'no',
    'ChallengeResponseAuthentication' => 'no',
    'PasswordAuthentication' => 'yes',
    'X11Forwarding' => 'yes',
    'X11DisplayOffset' => '10',
    'PrintMotd' => 'no',
    'PrintLastLog' => 'yes',
    'TCPKeepAlive' => 'yes',
    'AcceptEnv' => 'LANG LC_*',
    'UsePAM' => 'yes',
  }
  # Client parameters
  $client_config_ensure           = 'present'
  $client_package_ensure          = 'present'
  $client_configuration_data      = [ { '*' => {} } ]

  case $::osfamily {
    'Redhat': {
      $client_config_path           = '/etc/ssh/ssh_config'
      $client_package_name          = 'openssh-clients'

      $server_config_path           = '/etc/ssh/sshd_config'
      $server_package_name          = 'openssh-server'
      $server_service_name          = 'sshd'
      $server_os_configuration_data = {
        'HostKey'   => [],
        'Subsystem' => 'sftp /usr/libexec/openssh/sftp-server /usr/lib/openssh/sftp-server',
      }
    }
    'Debian': {
      $client_config_path           = '/etc/ssh/ssh_config'
      $client_package_name          = 'openssh-client'

      $server_config_path           = '/etc/ssh/sshd_config'
      $server_package_name          = 'openssh-server'
      $server_service_name          = 'ssh'
      $server_os_configuration_data = {
        'HostKey'   => [ '/etc/ssh/ssh_host_rsa_key', '/etc/ssh/ssh_host_dsa_key', '/etc/ssh/ssh_host_ecdsa_key' ],
        'Subsystem' => 'sftp /usr/lib/openssh/sftp-server /usr/lib/openssh/sftp-server',
      }
    }
    default: {
      fail("${::module} is unsupported on ${::osfamily}")
    }
  }

}
```

The final piece we need to build out is the template.  This has the
two hashes in it, and for each of the parameters it checks to see if
the values are an array, if so it repeats the parameter name and then
the entry for each element of the array.  This means we can handle
multiple hostkeys.

```
##
## Managed by Puppet
##

<% @configuration_data.sort.each do |parameter, value|    -%>
<%   if value.is_a?(Array)                                -%>
<%     value.each do |v|                                  -%>
<%= parameter %> <%= v %>
<%     end                                                -%>
<%   else                                                 -%>
<%= parameter %> <%= value %>
<%   end                                                  -%>
<% end                                                    -%>

# OS specific entries.
<% @os_configuration_data.sort.each do |parameter, value| -%>
<%   if value.is_a?(Array)                                -%>
<%     value.each do |v|                                  -%>
<%= parameter %> <%= v %>
<%     end                                                -%>
<%   else                                                 -%>
<%= parameter %> <%= value %>
<%   end                                                  -%>
<% end                                                    -%>
```

###Individual user ssh configuration

This piece of the module is almost the same as the client
configuration, but we need to be able to take in a user name to build
out the appropriate configuration.  As a result this makes sense to
write as a define instead of a class.

We only need a few parameters here and we make the assumption that
users live in /home, but you can see that the only real difference
is that we allow the owner/group to be set for the file.  We reuse
the same template as the previous ssh::client::config class.

```puppet
define ssh::user(
    $ensure             = present,
    $owner              = $name,
    $group              = $name,
    $configuration_data = [ { '*'  => {} } ]
  ) {

    file { "${name}_ssh":
      ensure => directory,
      path   => "/home/${name}/.ssh/",
      owner  => $owner,
      group  => $group,
      mode   => '0700',
    } ->

    file { "${name}_ssh_config":
      ensure  => $ensure,
      path    => "/home/${name}/.ssh/ssh_config",
      owner   => $owner,
      group   => $group,
      mode    => '0600',
      content => template('ssh/ssh_config.erb'),
    }

  }
```

###Documentation

At the end of all this work on the module itself we should be sure that we 
appropriately document it.  We have a README template available on our (website)[whereisthereadme]
which you should use as the basis of your documentation as it contains all the
appropriate sections to fill out.  In our case we'll document out the classes
and defines we made.

```
#ssh

####Table of Contents

1. [Overview](#overview)
2. [Module Description - What the module does and why it is useful](#ssh-description)
3. [Setup - The basics of getting started with [Ssh]](#setup)
    * [What [Ssh] affects](#what-[ssh]-affects)
    * [Setup requirements](#setup-requirements)
    * [Beginning with [Ssh]](#beginning-with-[Ssh])
4. [Usage - Configuration options and additional functionality](#usage)
5. [Reference - An under-the-hood peek at what the module is doing and how](#reference)
5. [Limitations - OS compatibility, etc.](#limitations)
6. [Development - Guide for contributing to the module](#development)

##Overview

This module manages OpenSSH's daemon, server configuration, client
configuration, and user specific client configuration.

##Module Description

This module manages OpenSSH and the associated configuration with both server
and clients.  It'll also allow you to configure per user configuration.

##Setup

###What [Ssh] affects

* sshd_config
* ssh_config
* ~/.ssh/ files.
* sshd daemon.

###Beginning with [Ssh]

In order to ensure sshd is running you just need to:

```
include ssh::server
```

##Usage

The two main classes for this module are `ssh::server` and `ssh::client`.
Through these two classes you should be able to manage ssh comprehensively.

##Reference

###Classes

* ssh::client          - Main class for managing the ssh client global settings.
* ssh::client::install - Installs the main ssh client.
* ssh::client::config  - Configs the main ssh client.
* ssh::server          - Main class for managing the ssh server.
* ssh::server::install - Installs the ssh server.
* ssh::server::config  - Configures the ssh server.
* ssh::server::service - Configures the ssh service.
* ssh::params          - Default parameters.

###Parameters

###ssh::server

####`config_ensure`
Should the config file be present or absent.

####`config_path`

The path for the configuration file.

####`package_ensure`

Should the package be present or absent.

####`package_name`

What is the name of the ssh server package?

####`service_ensure`

Should be the service be running or stopped.

####`service_name`

What is the name of the ssh server service?

####`configuration_data`

This allows you to pass in arbitary configuration to the sshd_config file. You
create it in the format of:

{ 'Parameter' => 'Value' }

To change any individual value in this hash you'll need to copy the entire
hash from the params.pp file and modify it to taste.

####`os_configuration_data`

Similar to the `configuration_data` parameter this is a hash containing
configuration data that differs between operating systems.  This seperation
allows you to set only OS specific configuration settings without touching the
main hash if you so wish.

###ssh::client

####`config_ensure`

Should the config file be present or absent.

####`config_path`

The path for the configuration file.

####`package_ensure`

Should the package be present or absent.

####`package_name`

What is the name of the ssh client package?

####`configuration_data`

This parameter allows you to create configuration entries for multiple
hosts.  You must create an array of hashes, containing subhashes, such as:

[ { '*' => { 'Port' => '2222' }, 'testhost' => { 'Port => '22' } } ]

###Defines

* ssh::user

###Parameters

###ssh::user

####`ensure`

Should the config file be present or absent.

####`owner`

Who should own the configuration file.

####`group`

Which group should own the configuration file.

####`configuration_data`

This parameter allows you to create configuration entries for multiple
hosts.  You must create an array of hashes, containing subhashes, such as:

[ { '*' => { 'Port' => '2222' }, 'testhost' => { 'Port => '22' } } ]


##Limitations

This module only supports the following osfamily:

* Debian
* RedHat

##Development
```


