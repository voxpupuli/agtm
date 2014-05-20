#Advanced Guide to Modules: Types & Providers

##Introduction

Types and providers are the heart of how Puppet works.  They allow you to
express the "how to do something" through raw commands, outside of the
constraints of the Puppet DSL.  With that information Puppet can then figure
out how to transition from what you have to what you want.   They are written
in Ruby, like Puppet.

Types and providers are preferred over execs and definitions due the ability to
do things difficult within the Puppet DSL.  It's not possible to read the
results of an exec{} and do different things based on that output.  They are
also ideally suited for other forms of complex logic that is difficult to
manage within the constraints of the DSL.  For instance,  a provider that

An example of this is a provider that manages local redis instances.  It would
need to make an actual TCP connection to the database to retrieve the current
state of the properties you wish to manage with Puppet.  This would be impossible
within the DSL and therefore modeled as a type and provider.

To highlight these practices, we will expand the basic ssh module we built in
the [previous section](link to Classes, Params, and APIs) to add a type and
provider to support the creation of ssh tunnels, as well as SOCKS proxies.

###Types & Providers vs. Defines

It would not be unusual to model something like ssh tunnels or SOCKS proxies
within a definition in Puppet to allow the creation of multiple instances.  While
this may work in the most simplistic of cases it usually breaks down in production
quality modules due to the complexities of managing the underlying system state
with the limitations of `exec{}`.  You cannot do sophisticated logic easily and
are limited to chaining together `exec{}` resources to try and do various things
based on exit codes and shell commands.

To most effectively model ssh tunnels and SOCKS proxies using types and
providers, we must be able to:

* Create an SSH tunnel based on various parameters.
* Validate the parameters to make sure they conform to the needs of the command
  that runs to build the tunnel.
* Test if a tunnel is running and only start it if not.
* Close an SSH tunnel if `ensure => absent` is set.

Because `exec{}` resources are just simple wrappers for shell
commands, it would be extremely complex to model the workflow of an
SSH tunnel and SOCKS proxy using one. In order to successfully use an
`exec{}` resource in this case, all of the logic of closing tunnels,
testing the processes, and tracking the existing processes within
Puppet would need to be modeled within the `definition`. The logic
model would require many `exec{}` statements and custom facts to try
and query the existing state of the machine, as there is no easy way
within Puppet to use the results of an `exec{}` statement within
another `exec{}` statement.  This is because Puppet is intended to
declare the end state, not the steps required to reach the end state.
That's the entire purpose of a type and provider, to handle the step
by step actions to transition between states.

To provide a concrete example of what this might look like, here is an
example `unless` property of a hypothetical exec{} to manage ssh tunnels.

```puppet
unless => ‘ps auxww | grep “-L ${local_port}:${forward_host}:${forward_port}”’`
```

This would attempt to stop duplicate tunnels being started.  It has a
number of problems, being unable to handle systems that don't take
'auxww' as arguments to ps. You’d also have to ensure you handle -D as
well for SOCKS proxies, and this would make the logic complex.

This also wouldn't work if you wanted to change one of the properties of the
definition.  If you changed the port you passed into the resource it would be
unable to find the previous tunnel anymore thanks to the change in the grep
string.  It would then create a second tunnel.  You would have to keep every
tunnel definition you created, changing the ensure parameter to absent, and
never reuse names to ensure it correctly managed them all.

In addition to the above problems the validation of the parameters
would have to be done with various functions within the manifest.
None of this is impossible, but you’d end up with a difficult to reuse
define.  Moreover, it would be tied very specifically to the
implementation and it would be difficult to extend it for various
different systems and use cases.

As you’re reading the types and providers section you can correctly
infer that we recommend the use of types and providers for this kind
of functionality [link-to-start].  Types and providers open up the
full power of Ruby on the agent, allow for multiple providers to serve
as the backends to a single type, and moreover, allow you to control
resources directly from the CLI with `puppet resource` or from an
orchestration system like MCollective.  In the ssh module, using types
and providers allows us to easily audit SSH tunnels across the entire
infrastructure, while also opening up more sophisticated use cases
than simple execs{} can generate.

NOTE (ideally this should be in a box of some kind):
[Types and Providers](http://www.amazon.com/Puppet-Types-Providers-Dan-Bode/dp/1449339328?tag=88fd3e53f8c9c-20),
is the best reference material available for writing types and
providers.  It covers all the pieces of functionality we’ll use below
in vastly more detail and is an invaluable guide.


###Intro to SSH Tunnels

We’re going to extend our ssh module by adding a type and provider capable of
managing ssh tunnels.  If you’re not familiar with the concept, an ssh tunnel
allows you to tunnel TCP/IP traffic from a local port through a remote server
to a destination.  An example of this might be:

```
ssh -L 8080:localhost:80 user@remoteserver
```

Running `ssh -L` would send any traffic from 8080 on your machine to the local
port 80 on user@remotewebserver. Or, rather than localhost, you could use
another hostname to have ‘remoteserver’ act as a kind of proxy for your
traffic.  You can even use ssh tunneling to set up a SOCKS proxy in order to
forward traffic of all kinds:

```
ssh -D 8080 user@remoteserver
```

For further reading on SSH tunnels you can [TODO: find a resource]

##Type

Types represent the resource description.  They list out all the
parameters that are required, their validation, any munging (forcing
passed in parameters to conform to certain needs, like forcing
everything to be lowercase.), or special casing of how the parameters
discovered on the system are compared to the required ones.  They are
written in Ruby and placed in lib/puppet/type/type_name.rb. 

In our ssh module, we’re going to write `ssh_tunnel`, a type that sets
up tunnels.  We have two use cases we want to target for now, one is:

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

Type declarations declare the name of the resource we're creating.
Best practices recommend that you try to keep the name short and
sweet.

In our ssh module, we begin with a simple type declaration at the
top. To write your own types, you just need to change ':ssh_tunnel' to
':type_name' and then update @doc to contain an appropriate
documentation string.

```
Puppet::Type.newtype(:ssh_tunnel) do
  @doc = 'Manage SSH tunnels'
```

Type validations ACCOMPLISH X and SHOULD/MUST BE USED IN Z WAY/FOR Y PURPOSE.
Validation has two sorts (I will fix that word later): regular  and global.
Regular validation, referred to as simply 'validation', does/is used to X.
Global validation does/is used to Y. Best practices HAS SOME THOUGHTS ABOUT
WHAT'S BEST.

Type validations allow you to verify that the contents of your
resource parameters conform to various validation rules.  There are
two sorts of validation, parameter and global.  Parameter validation,
referred to as simply 'validation' is for inspecting and verifying the
contents of a single parameter.  Global validation can be used to
ensure that multiple parameters conform.  An example might be making
sure that two conflicting parameters are never set together.  Best
practices recommends validating as much of your potential input as
possible.

First we'll look at some global validation in our ssh module. We do
this by calling the validate method and passing it a block of things
to test.  In our type we just need to make sure that `forward_server`
and `forward_port` are set if we’re not in SOCKS mode.

If either of these conditions aren’t true, we call fail() with a
reason.

```
  validate do
    # If we're not in SOCKS mode, we need a server and port to forward to.
    if self[:socks] == :false
      fail('forward_server is not set') if self[:forward_server].nil?
      fail('forward_port is not set') if self[:forward_port].nil?
    end
  end
```

One benefit to writing a type and provider is that there are various
helper methods that make common tasks easier.  When we call ensurable
it'll automatically create an ensure property with appropriate
validation handling.

```
  # This automatically creates the ensure parameter.
  ensurable
```

Next, we create our first “parameter”.  Puppet distinguishes between
properties and parameters, and the difference is just that properties
can be managed and discovered from the system, whereas parameters
aren’t discoverable and can’t be managed. [COMMENT] In our type the name is
really not something you “manage” on the local system, it’s just a
reference point for Puppet.  That means it’s a parameter, not a
property.

In order to allow this type to work with `puppet resource` we’ll need
to define some way for Puppet to automatically name discovered
resources.  The provider we create after the type will be able to
discover the existing instances that were created on the machine, such
as:

```
ssh -L 8080:localhost:80 user@remote
```

It'll then translate that into an appropriate name value such as:

```
ssh_tunnel { ‘8080:localhost:80-user@remote’: }
```

This is a little clumsy but we need the name to be unique, and
changing any of the information about the tunnel means it’s a
completely new one.  For other kinds of resources it’s easier to
provide a short and memorable name, as only the properties change and
there’s a static name to refer to (like package{}).

```
  newparam(:name, :namevar => true) do
    desc "The SSH tunnel name, must be unique."
  end
```

Our first property introduces ‘newvalues()’ and ‘defaultto()’.  These
methods are some of the helpers Puppet contains to make it easier to
validate input and provide defaults.  The first method, newvalues(),
accepts a comma seperated list of values to accept.  These can be
symbols, strings, or even regular expressions.  It’s fine to have
newvalues(/^\//) in order to enforce an absolute path.  The second
method, defaultto() just specifies the default value that Puppet
should set if you don’t pass the parameter in when defining the
resource in your manifest.

```
  newproperty(:socks) do
	desc 'Should this be a SOCKS proxy'
	newvalues(:true, :false)
	defaultto :false
  end
```

Our next property demonstrates an alternative, more sophisticated, use
of validation.  In validate we’re binding the value passed into
validate to the variable ‘port’, making it available for checking
within.  If you were to pass an array to local_port it would iterate
over that array and pass each element to validate. [COMMENT]

Inside the block you can do whatever ruby code you like.  We’re trying
to verify that port can be converted to an integer and then tested to
make sure it’s a valid port range.

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

The rest of our properties are all similar to the above explanations,
so we won’t explain these further [COMMENT].

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
.provide(:ssh) added.  We also set up a command.  Behind the scenes
this finds appropriate executables within the system path.  It does so
in a generic, cross-operating-system way so this works for Linux or
Windows [COMMENT]:

```
Puppet::Type.type(:ssh_tunnel).provide(:ssh) do
  desc 'Create ssh tunnels'

  commands :ssh => 'ssh'
```

`instances` is the heart of a provider.  It’s the method that Puppet
runs to discover all the existing instances of the thing the provider
is managing.  In the case of `user{}` that would be all the users.  In
our case it means that it needs to check `ps` and find all running
tunnels.  We do this by calling a method we haven't written yet,
`ssh_processes`.  This'll take a string to search for and attempt to
discover it on the server.  We'll write this later.

Once we've discovered each of the processes we'll split the string into
various pieces and use those to call new() which all the properties that
were returned from ssh_processes.  We'll build an array of all these
new() calls and return that to Puppet as the list of instances we found.

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

Prefetch is called the first time Puppet encounters a resource in the
catalog that matches the type that the provider is written for.  This
allows you to run certain logic before any resource of that type is
applied.  Maybe you need to run a command that temporarily allows
changes to be applied to the thing being
managed.  

The most common use for `prefetch` is shown below.  When Puppet calls
`prefetch` it passes in a hash of all the resources of the matching
type.  This can be used to set this provider as managing each of the
instances we discover on the system.

This is difficult to explain and complex to understand.  For now you
can simply cut and paste this into providers and tweak it a little to
be appropriate to speed things up.

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

Puppet runs `exists?` to determine if a resource actually exists on
the agent.  This allows you to determine the existence of a resource
in cases where it's not practical to create a list of all of the
resources on the agent with `instances`.  Files would be an example
of this, it would be expensive to create a resource for every file
on the system through `instances`.

If Puppet contains a resource in the catalog with an ensure property
set to present and can't find it as existing via `exists?` it'll run
`create` to call the appropriate logic to create the resource.

In our `create` method we’re checking if the @resource[:socks] (aka
socks => x in the resource definition) is true.  If it’s true then we
call ssh() with appropriate parameters to create a socks proxy.  If
@resource[:socks] is false then we just create a regular tunnel.

ssh() is created by the earlier `commands ssh => :ssh` statement.  The
reason for this rather than calling commands directly is to add some
additional features like showing calls via command() when you use
--debug, automatic confining of the provider to only systems that
contain the commands, and consistent command failure handling.

You pass in arguments to ssh() one at a time, separated by spaces, and
it safely builds up the appropriate command string to use.  In our
case we’re mostly just passing in other bits of information from
@resource, which is full of the values from the defined resource in a
manifest.

By setting the @property_hash[:x] values to the @resource[:x] value,
our provider is able to automatically create ‘getters’ (methods that
get various bits of information) later in the code.

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

Destroy is, unsurprisingly, the method Puppet runs if ensure => absent
and `exists?` returned true.  In this case it simply checks the pid
value contained within the @property\_hash and sends it a SIGTERM to
shut it down.  Once it does that it clears out the @property_hash to
reflect that the resource, and all its properties, is gone.

```
  def destroy
    Process.kill('SIGTERM', @property_hash[:pid].to_i)
    @property_hash.clear
  end
```

Here is how Puppet determines if a resource already exists.  It just simply
checks the property\_hash we've built up when running `instances` and `create`
to see if the resource is present.  If we had not written an `instances` method
due to it being too expensive to run on the agent, we'd have had to check ps
directly within `exists?` to determine if this resource was on the agent.

```
  def exists?
    @property_hash[:ensure] == :present || false
  end
```

Here is a piece of ruby metaprogramming that creates other methods within the
provider.  It understands the properties and parameters contained within the
type definition and automatically creates `property` and `property=` for each
discovered from the type.   By default these check the @property_hash to find
the appropriate value for the property.

If you find that `mk_resource_methods` is appropriate for most of your methods
but you sometimes need to tweak one of them (perhaps a tunnel=) then you can
simply define another method right after this call to override the generated
one.

```
  mk_resource_methods
```

Here we have a helper method that passes the appropriate ssh_opts in
to ssh() commands so that we don’t have to remember to add them
everywhere. We create it as self.ssh_opts to make it a class method
rather than an instance method, meaning there's just a single method
that lives in the class rather than each instance of this provider.

If this sounds confusing to you then you can read more about this on
various ruby sites or in ruby books.  An example of this can be found
at
(rubymonks)[http://rubymonk.com/learning/books/4-ruby-primer-ascent/chapters/45-more-classes/lessons/113-class-variables].

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

Beyond this we wish to make the methods in our provider private so
they cannot be accidentally called from outside of the provider itself.
Any method defined below this word will be considered private.

```
  private
```

Last, but not least, we have the `ssh_processes` method that actually
determines what tunnels are running on the agent.  It calls `ps` and
attempts to use regular expressions to extract the appropriate pid and
process details needed to populate the instances in self.instances().

One of the difficulties with writing types and providers is avoiding
dependencies on ruby libraries that aren't already part of the
standard ruby distribution, or dependencies of Puppet.  We could have
written code here that called out to any of the process handling
libraries of ruby, which would have been far more portable.  However,
that would mean the ssh module would need to distribute or install
this library on all systems and sometimes that's not desired.

This is why the code below manually runs and processes the output of
`ps`.  It's unportable to Windows and systems with a different `ps`
command.  For real world usage we would want to consider finding
better ways to handle this.

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
