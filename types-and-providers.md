#Advanced Guide to Modules: Types & Providers

##Introduction

Types and providers are the heart of how Puppet works.  They allow you to
express the "how to do something" through raw commands, outside of the
constraints of the Puppet DSL.  With that information Puppet can then figure
out how to transistion from what you have to what you want.   They are written
in Ruby, like Puppet.

Types and providers are preferred over execs and definitions due the ability to
do things difficult within the Puppet DSL.  It's not possible to read the
results of an exec{} and do different things based on that output.  They are
also ideally suited for other forms of complex logic that is difficult to
manage within the constraints of the DSL.  For instance,  a provider that

An example of this is a provider that manages local redis instances.  It would
need to make an actual TCP connection to the database to retrieve the current
state of the properties you wish to manage with Puppet.  This would be impossible
within the DSL and therefore modelled as a type and provider.

To highlight these practices, we will expand the basic ssh module we built in
the [previous section](link to Classes, Params, and APIs) to add a type and
provider to support the creation of ssh tunnels, as well as SOCKS proxies.

###Types & Providers vs. Defines

It would not be unusual to model something like ssh tunnels or SOCKS proxies
within a `define` in Puppet, to allow the support of more than one.  However,
this rarely works in practice because of the complexities of managing the
underlying system state is difficult to define with only Exec.  You cannot do
sophisticated logic easily, being limited to exit code checking and shell
commands. 

To most effectively model ssh tunnels and SOCKS proxies using types and
providers, we must be able to:

* Create an SSH tunnel based on various parameters.
* Validate the parameters to make sure they conform to the needs of the command that ran to build the tunnel.
* Test if a tunnel is running and only start it if not.
* Close an SSH tunnel if `ensure => absent` is set.

Because `exec{}` resources are just simple wrappers for shell commands, it
would be extremely complex to model the workflow of an SSH tunnel and SOCKS
proxy using one. In order to successfully use an `exec{}` resource in this
case, all of the logic of closing tunnels, testing the processes, and tracking
the existing processes within Puppet would need to be modeled within the
`define`. The logic models would require many `exec{}` statements and facts to
try querying the existing state of the machine, since there is no way within
Puppet to run an exec and use the results in another execYou'd need many exec{}
statements as well as facts in order to try and query the existing state of the
machine (as there's no way within Puppet to run an exec and then use the
results in another exec) because we can only declare the end state, not the
method of getting there (which is what a t&p does, it does all the getting
there steps) Does that help?

`unless => ‘ps auxww | grep “-L ${local_port}:${forward_host}:${forward_port}”’`

in an attempt to stop duplicate tunnels being started.  You’d have to ensure
you handle -D as well for SOCKS proxies, and this would make the logic complex.

In addition this wouldn’t work in the face of a slightly tweaked resource.  If
you simply changed the port you passed in it would create a second, separate,
tunnel.  You would have to keep the original tunnel definition, change ensure
=> to absent, and then create the new one.

The validation of the parameters would have to be done with various functions
within the manifest.  None of this is impossible, but you’d end up with a
difficult to reuse define.  Moreover, it would be tied very specifically to the
implementation and it would be difficult to extend it for various different
systems and use cases.

As you’re reading the types and providers section you can correctly infer that
we recommend the use of types and providers for this kind of functionality
[Change this to point back to the beginning of this section: We strongly
recommend using types and providers for X] .  Types and providers open up the
full power of Ruby on the agent, allow for multiple providers to serve as the
backends to a single type, and moreover, allow you to control resources
directly from the CLI with `puppet resource` or from an orchestration system
like MCollective.  In the ssh module, using types and providers  allows us to
easily audit SSH tunnels across the entire infrastructure, while also opening
up more sophisticated use cases than simple execs{} can generate.

NOTE (ideally this should be in a box of some kind): [Types and
Providers](http://www.amazon.com/Puppet-Types-Providers-Dan-Bode/dp/1449339328?tag=88fd3e53f8c9c-20),
is the best reference material available for writing types and providers.  It
covers all the pieces of functionality we’ll use below in vastly more detail
and is an invaluable guide.


###Intro to SSH Tunnels

We’re going to extend our ssh module by adding a type and provider capable of
managing ssh tunnels.  If you’re not familiar with the concept, an ssh tunnel
allows you to tunnel TCP/IP traffic from a local port through a remote server
to a destination.  An example of this might be:

```
ssh -L 8080:localhost:80 user@remoteserver
```

Running `ssh -L` would send any traffic from 8080on your machine to the local
port 80 on user@remotewebserver. Or, rather than localhost, you could use
another hostname to have ‘remoteserver’ act as a kind of proxy for your
traffic.  You can even use ssh tunneling to set up a SOCKS proxy in order to
forward traffic of all kinds:

```
ssh -D 8080 user@remoteserver
```

SSH tunnels can get weird quickly. There are some great guides/references HERE

##Type

Types are the code that represents the resource description.  They list out all
the parameters that are required, their validation, any munging (forcing passed
in parameters to conform to certain needs, like forcing everything to be
lowercase.)  They are written in Ruby and placed in
lib/puppet/type/type_name.rb. Best practices recommend using types
TO/AS/FOR/SOMETHING.

In our ssh module, we’re going to write `ssh_tunnel`, a type that sets up
tunnels.   We have two use cases we want to target for now, one is:

```
ssh_tunnel { ‘bypass firewall’:
  ensure         => present,
  local_port     => ‘8080’,
  forward_server => ‘localhost’,
  forward_port   => ‘80’,
  remote_server  => ‘remoteserver’,
}
```

As well as the SOCKS proxy option:

```
ssh_tunnel { ‘socks proxy’:
  ensure        => present,
  local_port    => ‘8080’,
  socks         => true,
  remote_server => ‘remoteserver’,
}
```

This means that our required parameters are `remote_server`, `local_port`, and
`ensure`.  Other parameters are optional depending on which mode we’re using.

###Demonstration

Type declarations ARE ZZZ and ARE USED FOR/TO. Best practices recommend/state
you should/must do X AND Y AND Z when declaring types. 

In our ssh module, we begin with a simple type declaration at the top. To write
your own types, you just need to change ':ssh_tunnel' to ':type_name' and then
update @doc.

```
Puppet::Type.newtype(:ssh_tunnel) do
  @doc = 'Manage SSH tunnels'
```
X



Type validations ACCOMPLISH X and SHOULD/MUST BE USED IN Z WAY/FOR Y PURPOSE.
Validation has two sorts (I will fix that word later): regular  and global.
Regular validation, referred to as simply 'validation', does/is used to X.
Global validation does/is used to Y. Best practices HAS SOME THOUGHTS ABOUT
WHAT'S BEST.

First we'll look at  some global validation in our ssh module. X  We do this by
calling the validate method and passing it a block of things to test.  In our
type we just need to make sure that `forward_server` and `forward_port` are set
if we’re not in SOCKS mode.

If either of these conditions aren’t true, we call fail() with a reason. [Intro
to what this code is, what it's doing, and where it goes.]

```
  validate do
    # If we're not in SOCKS mode, we need a server and port to forward to.
    if self[:socks] == :false
      fail('forward_server is not set') if self[:forward_server].nil?
      fail('forward_port is not set') if self[:forward_port].nil?
    end
  end
```

The nice thing about writing a type/provider vs a definition within a manifest
is that Puppet has a bunch of helper code to make certain common things easier.
For instance, calling the ‘ensurable’ method gives us automatic 'ensure =>
present/absent' handling with automatic validation. Puppet also knows how to
use various methods we’ll define in the provider to test for the presence of
the resource on the local system so as to make sure it won’t do anything if the
resource is correct.


```
  # This automatically creates the ensure parameter.
  ensurable
```

Next, we create our first “parameter”.  Puppet distinguishes between properties
and parameters, and the difference is just that properties can be managed and
discovered from the system, whereas parameters aren’t discoverable and can’t be
managed.  In our type the name is really not something you “manage” on the
local system, it’s just a reference point for Puppet.  That means it’s a
parameter, not a property.

In order to allow this type to work with `puppet resource` we’ll need to define
some way for Puppet to automatically name discovered resources.  [Therefore, we
must DO X ACTION. X ACTION accomplishes Y by Z.] This means that Puppet will
take something like:

```
ssh -L 8080:localhost:80 user@remote
```

And give it an appropriate name like:

```
ssh_tunnel { ‘8080:localhost:80-user@remote’: }
```

This is a little clumsy but we need the name to be unique, and changing any of
the information about the tunnel means it’s a completely new one.  For other
kinds of resources it’s easier to provide a short and memorable name, as only
the properties change and there’s a static name to refer to (like package{}).

```
  newparam(:name, :namevar => true) do
    desc "The SSH tunnel name, must be unique."
  end
```

Our first property introduces ‘newvalues()’ and ‘defaultto()’.  These methods
are some of the helpers Puppet contains to make it easier to validate input and
provide defaults.  The first method, newvalues(), accepts a comma seperated
list of values to accept.  These can be symbols, strings, or even regular
expressions.  It’s fine to have newvalues(/^\//) in order to enforce an
absolute path.  The second method, defaultto() just specifies the default
Puppet should assume if you don’t pass the parameter in when defining the
resource in your manifest.

```
newproperty(:socks) do
    desc 'Should this be a SOCKS proxy'
    newvalues(:true, :false)
    defaultto :false
  end
```

Our next property demonstrates an alternative, more sophisticated, use of
validation.  In validate we’re binding the value passed into validate to the
variable ‘port’, making it available for checking within.  If you were to pass
an array to local_port it would iterate over that array and pass each element
to validate.

Inside the block you can do whatever ruby code you like.  We’re trying to
verify that port can be converted to an integer and then tested to make sure
it’s a valid port range.

```
  newproperty(:local_port) do
    desc 'The local port to forward traffic from'
    validate do |port|
      fail("Port is not in the range 1-65535") unless port.to_i >= 1 and port.to_i <= 65535
    end
  end
```

Afterwards this results in output such as:

```
# puppet resource ssh_tunnel 8080-test@localhost local_port=65536
Error: Could not run: Parameter local_port failed on Ssh_tunnel[8080-test@localhost]: Port is not in the range 1-65535
```

The rest of our properties are all similar to the above explanations, so we
won’t explain these further.

```
  newproperty(:forward_port) do
    desc 'Port to forward traffic to'
    validate do |port|
      fail("Port is not in the range 1-65535") unless port.to_i >= 1 and port.to_i <= 65535
    end
  end

  newproperty(:remote_server) do
    desc 'Remote server to connect to'
    newvalue(/\w+/)
  end

  newproperty(:forward_server) do
    desc 'Server to forward traffic to'
    newvalue(/\w+/)
  end
end
```

#Provider

Once the type has been written you need a provider to ‘provide’ the services to
the type.  In this case the type is tied specifically to the provider, but we
could have tried to make a generic “tunnel” type that took ssh and other
providers.  That’s good practice when feasible, but it’s ok to make a specific
type/provider as well when it makes sense.  Don’t over abstract before you know
abstraction has a benefit.

#Demonstration

We start our provider by creating the Puppet::Type definition with
.provide(:ssh) added.  We also set up a command.  This call to commands does a
whole bunch of automatically checking of paths and so forth to make sure it
finds the appropriate file in a generic, cross distribution, way:

```
Puppet::Type.type(:ssh_tunnel).provide(:ssh) do
  desc 'Create ssh tunnels'

  commands :ssh => 'ssh'
```

`instances` is the heart of a provider.  It’s the method that Puppet runs to
discover all the existing instances of this provider on your system.  In our
case it means that it needs to check `ps` and find all running tunnels.  Once
it does this it creates a resource for each (the instances << new() bit) and
returns them all at the end to Puppet.

```
  def self.instances
    instances = []
    tunnel_connections = ssh_processes("#{ssh_opts} -L")
    socks_connections  = ssh_processes("#{ssh_opts} -D")

    tunnel_connections.each do |name, pid|
      rest, remote_host = name.split(' ')
      local_port, forward_server, forward_port = rest.split(':')
      instances << new(
        :ensure         => :present,
        :name           => "#{rest}-#{remote_host}",
        :local_port     => local_port,
        :forward_server => forward_server,
        :forward_port   => forward_port,
        :remote_host    => remote_host,
        :socks          => :false,
        :pid            => pid
      )
    end

    socks_connections.each do |name, pid|
      local_port, remote_host = name.split(' ')

      instances << new(
        :ensure      => :present,
        :name        => "#{local_port}-#{remote_host}",
        :local_port  => local_port,
        :remote_host => remote_host,
        :socks       => :true,
        :pid         => pid
      )
    end

    return instances
  end
```

Prefetch is called the first time Puppet encounters a resource of the type the
provider is written for.  This lets you run code before any of the resources of
that type are applied.  The most common use for this is seen below, caching of
all the resources that exist on the system so it doesn’t have to figure out the
properties of each of them individually.  Below we call self.instances and set
the provider for each instance discovered to be the provider we’re in.  This is
actually fairly complex to understand, so don’t worry if it makes no sense on
first reading, you can just cut and paste self.prefetch into your other
providers.

```
  def self.prefetch(resources)
    # Obtain a hash of all the instances we can discover
    tunnels = instances
    resources.keys.each do |name|
      # If we find the resource in the catalog in the discovered list
      # then set the provider for the resource to this one.
      if provider = tunnels.find { |tunnel| tunnel.name == name }
        resources[name].provider = provider
      end
    end
  end
```

Create is the method run when a resource isn’t found by self.instances is
required due to ensure being set to present. This then creates the resource on
your server. 

In our method we’re checking if the @resource[:socks] (aka socks => x in the
resource definition) is true.  If it’s true then we’re calling ssh() with
appropriate parameters to create a socks proxy, else we create a tunnel.

ssh() is created by the earlier `commands ssh => :ssh` statement.  The reason
for this rather than calling commands directly is to add some additional
features like showing calls via command() when you use --debug, automatic
confining of the provider to only systems that contain the commands, and
consistent command failure handling.

You pass in arguments to ssh() one at a time, separated by spaces, and it
safely builds up the appropriate command string to use.  In our case we’re
mostly just passing in other bits of information from @resource, which is full
of the values from the defined resource in a manifest.

By setting the @property_hash[:x] values to the @resource[:x] value, our
provider is able to automatically create ‘getters’ for various bits of data
later in the code.

```
  def create
    if @resource[:socks] == :true
      # Create a SOCKS proxy
      ssh(ssh_opts, '-D', @resource[:local_port], @resource[:remote_server])
      @property_hash[:ensure]        = :present
      @property_hash[:local_port]    = @resource[:local_port]
      @property_hash[:remote_server] = @resource[:remote_server]
    else
      # Create an SSH tunnel
      ssh(ssh_opts, '-L', "#{@resource[:local_port]}:#{@resource[:forward_server]}:#{@resource[:forward_port]}", @resource[:remote_server])
      @property_hash[:ensure]         = :present
      @property_hash[:local_port]     = @resource[:local_port]
      @property_hash[:forward_server] = @resource[:forward_server]
      @property_hash[:forward_port]   = @resource[:forward_port]
      @property_hash[:remote_server]  = @resource[:remote_server]
    end

    exists? ? (return true) : (return false)
  end
```

Destroy is, unsurprisingly, the method Puppet runs if ensure => absent.  In
this case it simply extracts the pid it discovered in self.instances and sends
it a SIGTERM to shut it down.  Once it does that it clears out the
@property_hash to reflect that the resource, and all its properties are gone.

```
  def destroy
    Process.kill('SIGTERM', @property_hash[:pid].to_i)
    @property_hash.clear
  end
```

This is how Puppet determines if a resource already exists.  It just simply
checks the property_hash (as we populate that in all the other functions like
create() and instances(), otherwise this would have to check ps too) or returns
false if @property_hash[:ensure] isn’t present.

```
  def exists?
    @property_hash[:ensure] == :present || false
  end
```

This method automatically creates methods that get the results of the property
hash.  You effectively get all your `def socks` and `def local_port` getters
without any additional work.  You can always combine this by redefining
specific methods right after if you need to be cleverer than just checking the
@property_hash value.

```
  mk_resource_methods
```

Here we have a helper method that passes the appropriate ssh_opts in to ssh()
commands so that we don’t have to remember to add them everywhere.  It’s added
as a self.ssh_opts so it’s visible to self.instances, and as ssh_opts so it’s
also visible within instances.  I found this confusing at first, but sites such
as
(RubyMonk)[http://rubymonk.com/learning/books/4-ruby-primer-ascent/chapters/45-more-classes/lessons/113-class-variables]
cover this in more detail.

```
  # -f tells ssh to go into the background just before command execution
  # -N tells ssh not to execute remote commands
  def self.ssh_opts
    '-fN'
  end
  # This allows ssh_opts to be used within the instance.
  def ssh_opts
    self.class.ssh_opts
  end
```

This makes all the the methods below this point in the code private to the
class.

```
  private
```

Here is the heart of our provider.  This method runs `ps` and attempts to regex
out the appropriate pid and process details so that we can populate the
instances in self.instances().

One of the largest difficulties in writing generic types and providers is
avoiding a dependency on ruby gems that aren’t already puppet dependencies,
unless you’re willing to be responsible for installing them via the manifests
that go with the module.  In our case this means we have to run ps and attempt
to process it.  This code is now unportable to Windows and possibly other
distributions with slightly different ps versions, and for real world usage we
would look for a better pattern for this.  We could check /proc/ for linux, for
example.

```
  # Find and return the ssh tunnel/proxy names and their associated pid.
  def self.ssh_processes(pattern)
    ssh_processes = {}
    ps = Facter["ps"].value
    IO.popen(ps) do |table|
      table.each_line do |line|
        # Attempts to match 'user (pid).*ssh {sshopts} (port:host:port host)'
        if match = line.match(/^\w+\s+(\d+)\s.*ssh\s#{pattern}\s(\d+:?\w*:?\d*\s\w.+)/)
          pid, name = match.captures
          ssh_processes[name] = pid
        end
      end
    end
    return ssh_processes
  end

end
```
