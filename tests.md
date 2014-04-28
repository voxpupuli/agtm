-> Test planning should be a doc before this.

#Introduction

Extending on from the three previous guides,
[Classes, Params, APIs](link), [Types and Providers](link), and
[Facts](link), we finally introduce testing into our SSH module.
Normally you would write tests as you go, rather than retrofit them in
at the end, and we recommend a BDD development system.

Having said that, we’re now going to retrofit in `puppet-rspec` unit
tests, rspec unit tests for the type/provider, rspec unit tests for
the fact, and Beaker acceptance tests to test the full workings of the
module.

For examples of all of the testing we’re going to do you can
investigate the puppetlabs modules
[on the forge](https://forge.puppetlabs.com).  Modules like MySQL,
Apache, RabbitMQ, PostgreSQL and Apt have all sorts of tests and you
can often steal entire chunks for use in your own testing.

#Manifest Testing

Rspec-puppet is the framework that we use for manifest testing.
Written and maintained by Tim Sharpe, better known as `rodjek`, a
community member and Githubber, rspec-puppet extends rspec to
understand Puppet and allows you to describe things in terms that
makes sense for manifests.

For rspec-puppet testing you want to cover your “entry” classes,
rather than having a spec test for the private subclasses.  That means
for our module we just need to test ssh::client and ssh::server.

Before we start the actual manifest testing we’ll cover setting up
your module to be testable:

##Setup

In order to test a module we need to do several things; add a Gemfile
with the testing framework dependencies, add a spec_helper.rb file, a
.fixtures.yml and a Rakefile.  Our Gemfile will be fairly barebones
and borrowed from the puppetlabs modules:

The opening section just sets the source of the gems we use, and
creates a “development” group for [Bundler](http://bundler.io/) to
consume later.

```
source 'https://rubygems.org'

group :development, :test do
```

This is our list of gems to install.  I won’t talk about all of them,
but ‘Rake’ is used for a kind of Makefile that allows us to type `rake
spec` to run tests, ‘puppetlabs_spec_helper’ adds some rake tests, and
the last two are for code coverage and puppet-lint runs via Rake.

```
  gem 'rake',                    :require => false
  gem 'rspec-puppet',            :require => false
  gem 'puppetlabs_spec_helper',  :require => false
  gem 'puppet-lint',             :require => false
  gem 'simplecov',               :require => false
end

if puppetversion = ENV['PUPPET_GEM_VERSION']
  gem 'puppet', puppetversion, :require => false
else
  gem 'puppet', :require => false
end

# vim:ft=ruby
```

With our Gemfile in place we can now add the Rakefile:

```
require 'puppetlabs_spec_helper/rake_tasks'
```

Then the .fixtures.yml file, which is used to download and setup any
dependencies when doing testing.

```
fixtures:
  repositories:
    "stdlib": "git://github.com/puppetlabs/puppetlabs-stdlib"
  symlinks:
    "ssh": "#{source_dir}"
```

And lastly our spec/spec_helper.rb.  We require and include simplecov as this
will generate code coverage reports for types and providers.  Sadly it doesn’t
work for manifests.  Lastly it includes puppetlabs_spec_helper which does all
the rest of the work for us:

```
require 'simplecov'
SimpleCov.start do
  add_filter "/spec/"
end
require 'puppetlabs_spec_helper/module_spec_helper'
```

At this point you should be able to issue `be rake spec`.  If this fails then
the rest of this section won’t work for you.  You can grab the module with all
the required files [at
github](https://github.com/apenney/puppetlabs-ssh/commit/6f7f2d978fc3d70412771cc210646f6db5225fe4)
to ensure there’s no issues in your local copy.

```
$ be rake spec
/opt/boxen/rbenv/versions/1.9.3-p448/bin/ruby -S rspec --color
No examples found.


Finished in 0.00009 seconds
0 examples, 0 failures
```

##Demonstration

Now we have the module ready for testing we can start by adding some simple
tests for ssh::client in the file spec/classes/ssh_client_spec.rb.  (All spec
tests should end in _spec.rb or the rake task won’t find them).

We open the file with a description of the class we want to test:

```
require 'spec_helper'

describe 'ssh::client' do
```

If you want to test multiple distributions or systems in rspec-puppet you have
to “mock” out the facts for that system to trick Puppet into supplying a
catalog for that system.  In order to do this in our tests we create a
‘shared_example’, which is basically a chunk of code you can pass parameters
into elsewhere in the test.  This lets us avoid repeating ourselves to test
multiple distributions.

Within this shared_example block we named ‘client’ we create two seperate
tests, one for the package, and one for the configuration file.  To make sure
our catalog works properly we’ll need to check that the package_name is right
and that the package_path is correct.  We use the variables in the
shared_examples block to fill in the gaps in the .with() sections of our tests.

```
  shared_examples 'client' do |osfamily, package_name, package_path|
    let(:facts) {{ :osfamily => osfamily }}

    it 'contains the package' do
      should contain_package('ssh').with(
        :ensure => 'present',
        :name   => package_name
      )
    end

    it 'contains the ssh_config file' do
      should contain_file('ssh_config').with(
        :ensure => 'present',
        :path   => package_path
      )
    end
  end
```

Now we call our ‘client’ shared_examples by passing variables into a method
called ‘it_behaves_like’.  We pass in the name of the shared_example first and
then the other variables in order.  This means that osfamily in the ‘client’
example gets get to ‘RedHat’ in the first example below.

```
  context 'RedHat' do
    it_behaves_like 'client', 'RedHat', 'openssh-clients', '/etc/ssh/ssh_config'
  end

  context 'Debian' do
    it_behaves_like 'client', 'Debian', 'openssh-client', '/etc/ssh/ssh_config'
  end
```

Here we test an unsupported operating system.  We set the fact to make the
osfamily something we didn’t account for in ssh::params and inside the it{}
block we state that “we expect (including the class) to raise an error()” which
it does, telling us that the class is unsupported.

```
  context 'Unsupported' do
    let(:facts) {{ :osfamily => 'Unsupported' }}
    it { expect { should contain_class('ssh::client') }.to raise_error(Puppet::Error, /is unsupported on Unsupported/) }
  end

end
```

Now we switch to testing the ssh::server class.  These tests are almost
identical to ssh::client except for the additional of a ‘contains_server()’
block in the main shared_examples.  However, we can extend our tests and add a
new block that tests the configuration parameters.

First we add a new section, let(:params) which takes a hash as the block body
with all the parameters for the class within.  We’ve set each configuration
parameter in the class to something specific:

```
  context 'template contents' do
    let(:facts) {{ :osfamily => 'Debian' }}
    let(:params) {
      { :port                    => '8080',
        :protocol                => '3',
        :permit_root_login       => false,
        :password_authentication => false,
        :sftp_path               => '/usr/test',
        :hostkeys                => [ 'key1', 'key2', 'key3' ]
      }
    }
```

Now we can do a series of checks on the file to make sure it contains the entry
we would expect.  You could condense this to a single regular expression that
tested everything at once but it’s not really a problem.  Intent and clarity is
more important than space saving in a test and this makes it very easy to add
new parameters by repeating the it{} block and tweaking the contents.

```

    it { should contain_file('sshd_config').with(
      :content => /Port 80/
    )}
    it { should contain_file('sshd_config').with(
      :content => /Protocol 3/
    )}
    it { should contain_file('sshd_config').with(
      :content => /PermitRootLogin false/
    )}
    it { should contain_file('sshd_config').with(
      :content => /PasswordAuthentication false/
    )}
    it { should contain_file('sshd_config').with(
      :content => /Subsystem sftp \/usr\/test/
    )}
    it { should contain_file('sshd_config').with(
      :content => /# HostKeys for protocol version 2\nkey1\nkey2\nkey3\n/
    )}
  end
```

###Results

When I ran the tests for the first time after writing them they failed with
errors like below:

```
  1) ssh::client RedHat behaves like client contains the package
     Failure/Error: )
       expected that the catalogue would contain Package[ssh] with ensure set to `"present"` but it is set to `nil` in the catalogue, and parameter name set to `"openssh-clients"` but it is set to `"ssh"` in the catalogue
     Shared Example Group: "client" called from ./spec/classes/ssh_client_spec.rb:24
     # ./spec/classes/ssh_client_spec.rb:12:in `block (3 levels) in <top (required)>'
```

It looked like ssh::client wasn’t picking up the information from ssh::params
properly and when I checked manifests/client.pp it turned out that I had
forgotten to inherit from ssh::params and it was indeed broken.

#Unit Testing

Now we have functioning manifest tests we switch to traditional rspec unit
testing.  We’re going to test the code we wrote for facts earlier.  Most module
writers are not traditional developers, and so we’ll build out our tests in a
slightly less sophisticated way than is supported by rspec in the interests of
clarity.  The module team has found that the tests for types, providers, and
facts, within Puppet can be terrible examples to learn from as they assume a
high level of familarity with ruby and with testing.  We hope the below
demonstration will be a bit more sysadmin oriented and easier to follow.

##Demonstration

We need two separate spec tests, one for each fact.   We start by including
`spec_helper` and `etc`.  Spec_helper is always included in our unit tests, but
etc is only required for access to Struct::Passwd, the
[struct](http://www.ruby-doc.org/core/Struct.html) that Etc.passwd returns when
called.
 
``` require 'spec_helper' require 'etc' ```

Here we use let() to introduce three things we need to mock up in our tests.
singleuser becomes an object made up of a Struct::Passwd containing a fake
entry from /etc/passwd, and multiuser becomes an array of two fake users.
Last, keys becomes a variable containing the contents of a fake id_rsa.pub
file.

```
describe 'public_key_comments', :type => :fact do
  let(:singleuser) { Struct::Passwd.new('test', nil, nil, 1, 1, '/home/test') }
  let(:multiuser) {[
    Struct::Passwd.new('test', nil, nil, 1, 1, '/home/test'),
    Struct::Passwd.new('test2', nil, nil, 2, 2, '/var/tmp/test')
  ]}
  let(:keys) {
    "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDVwwxl6dz6Y7karwyS8S+za4qcu99Ra8H8N3cVHanEB+vuigtbhLOSb+bk6NjxFtC/jF+Usf5FM5fGIYd51L7RE9BbzbKiWb9giFnNqhKWclO5CY4sQTyUyYiJTQKLuVtkmiFeArV+jIuthxm6JrdOeFx8lJpcgGlZjlcBGxp27EbZNGWIlAdvW0ZXy0JqS9M/vj71NBBDfkrpyzAPC0aBa9+FmywOH6HXbyeFooHLOw+mfzP87jwDDQ2yXIehDoC1BsLYXD+j+kdnR0CNltJh1PYOFNpbKQpfnPhfdw4Oc0hZ34n+kfBPavKlbwxoVAoisBWWo4c9ZnUoe2OBRHAX test@local
    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDQjmi7VZln/ehWW5nDgpHKWRSAOEUd//Qft00vqBq1khyGF6o0hUmLhjU1fxXv3w7GNshgoqTTsgMRNSOP06UaNJwU8g1Gyji9ard+mVtJ7ohgv/OSR2cujL/853Q/LOVo5LIgEKRxCyA1KAPE68n44WC3RIvUO78tk5flagAHYN/i+B8k4W040aqtEvTMjKGag7377eQIzp4GNJ8Hzm1VdFeZQQewAi/hrUOyU3gWoXTpN+xaWr41b4Vugrgb5V9/esDBXb+y2zj8Wc/hX2xc33crfWLkFh7YhmsgmhMEAKE8G2mZEG3Sx3/9BHNsleBTh0oJl5CZm+cBh+BCCGq/ test@127.0.0.1"
  }
```

Now we’re into the actual tests.  We start by describing what the test is for,
in this case a single user, then create our ‘before :each do’ block.  This is
code that is ran before each it{} within this describe block.  In our case we
start using ‘stubs’ as a way of faking the return of various functions that
exist in our fact.  Our first stub is for Etc.passwd, and we make it yield up
the singleuser struct we had before.  We next stub Dir[] with the test users
.ssh directory returning a single file, id_rsa.pub.  Last, we stub File.read()
for that file to return the fake keys content we defined earlier.

Inside the it{} block we then actually run Facter.fact() to call our real ruby
fact.  This code is run, but when the functions we stubbed are found it just
skips running them for real and returns the fake stuff we stubbed out.

We then use .value.should eq ‘thing’ to assert that our fact, when run with our
fake data, should find two comments.

```
  describe 'single user' do
    before :each do
      Etc.stubs(:passwd).yields(singleuser)
      Dir.stubs(:[]).with("/home/test/.ssh/*.pub").returns(["id_rsa.pub"])
      File.stubs(:read).with('/home/test/.ssh/id_rsa.pub').returns(keys)
    end

    it { Facter.fact(:public_key_comments).value.should eq 'test@local,test@127.0.0.1' }
  end
```

This test is almost identical to the previous one, except we stub out the
functions that are called for both the users found in the multiuser section.

```
  describe 'multiple user' do
    before :each do
      Etc.stubs(:passwd).yields(multiuser)
      Dir.stubs(:[]).with("/home/test/.ssh/*.pub").returns(["id_rsa.pub"])
      Dir.stubs(:[]).with("/var/tmp/test/.ssh/*.pub").returns(["id_rsa.pub"])
      File.stubs(:read).with('/home/test/.ssh/id_rsa.pub').returns(keys)
      File.stubs(:read).with('/var/tmp/test/.ssh/id_rsa.pub').returns(keys)
    end

    it { Facter.fact(:public_key_comments).value.should eq 'test@local,test@127.0.0.1' }
  end

end
```

Our tests for the second fact are extremely similar to our other test.  We
define the same let()s, the same describe blocks, the only differences come in
the stubbed functions.

```
require 'spec_helper'
require 'etc'

describe 'public_key_comments', :type => :fact do
  let(:singleuser) { Struct::Passwd.new('test', nil, nil, 1, 1, '/home/test') }
  let(:multiuser) {[
    Struct::Passwd.new('test', nil, nil, 1, 1, '/home/test'),
    Struct::Passwd.new('test2', nil, nil, 2, 2, '/var/tmp/test')
  ]}
  let(:keys) {
    "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDVwwxl6dz6Y7karwyS8S+za4qcu99Ra8H8N3cVHanEB+vuigtbhLOSb+bk6NjxFtC/jF+Usf5FM5fGIYd51L7RE9BbzbKiWb9giFnNqhKWclO5CY4sQTyUyYiJTQKLuVtkmiFeArV+jIuthxm6JrdOeFx8lJpcgGlZjlcBGxp27EbZNGWIlAdvW0ZXy0JqS9M/vj71NBBDfkrpyzAPC0aBa9+FmywOH6HXbyeFooHLOw+mfzP87jwDDQ2yXIehDoC1BsLYXD+j+kdnR0CNltJh1PYOFNpbKQpfnPhfdw4Oc0hZ34n+kfBPavKlbwxoVAoisBWWo4c9ZnUoe2OBRHAX test@local
    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDQjmi7VZln/ehWW5nDgpHKWRSAOEUd//Qft00vqBq1khyGF6o0hUmLhjU1fxXv3w7GNshgoqTTsgMRNSOP06UaNJwU8g1Gyji9ard+mVtJ7ohgv/OSR2cujL/853Q/LOVo5LIgEKRxCyA1KAPE68n44WC3RIvUO78tk5flagAHYN/i+B8k4W040aqtEvTMjKGag7377eQIzp4GNJ8Hzm1VdFeZQQewAi/hrUOyU3gWoXTpN+xaWr41b4Vugrgb5V9/esDBXb+y2zj8Wc/hX2xc33crfWLkFh7YhmsgmhMEAKE8G2mZEG3Sx3/9BHNsleBTh0oJl5CZm+cBh+BCCGq/ test@127.0.0.1"
  }

  describe 'single user' do
    before :each do
      Etc.stubs(:passwd).yields(singleuser)
      Dir.stubs(:[]).with("/home/test/.ssh/*.pub").returns(["id_rsa.pub"])
      File.stubs(:read).with('/home/test/.ssh/id_rsa.pub').returns(keys)
    end

    it { Facter.fact(:public_key_comments).value.should eq 'test@local,test@127.0.0.1' }
  end

  describe 'multiple user' do
    before :each do
      Etc.stubs(:passwd).yields(multiuser)
      Dir.stubs(:[]).with("/home/test/.ssh/*.pub").returns(["id_rsa.pub"])
      Dir.stubs(:[]).with("/var/tmp/test/.ssh/*.pub").returns(["id_rsa.pub"])
      File.stubs(:read).with('/home/test/.ssh/id_rsa.pub').returns(keys)
      File.stubs(:read).with('/var/tmp/test/.ssh/id_rsa.pub').returns(keys)
    end

    it { Facter.fact(:public_key_comments).value.should eq 'test@local,test@127.0.0.1' }
  end

end
```

#Types and Provider Testing

Testing for types and providers is similar to testing facts.  We still rely on
rspec, except this time we get a little more sophisticated with our stubbing
and creating of types and providers within the tests for behavior assertion.

##Demonstration

We start with our type test.  These are relatively straightforward because they
are generally about testing the validation of the class.  We require ‘puppet’
this time as we’re going to be creating Puppet::Types and testing them.

```
require 'spec_helper'
require 'puppet'

describe Puppet::Type.type(:ssh_tunnel) do
```

Here we create two instances of our type, one for socks and one for a tunnel, so we can easily test without repeating ourselves.

```
  let(:socks) { Puppet::Type.type(:ssh_tunnel).new(
    :name          => 'test',
    :local_port    => '8080',
    :socks         => :true,
    :remote_server => 'testserver'
  )}
  let(:tunnel) { Puppet::Type.type(:ssh_tunnel).new(
    :name           => 'test2',
    :forward_server => 'localhost',
    :forward_port   => '80',
    :local_port     => '8080',
    :socks          => :false,
    :remote_server  => 'testserver'
  )}
```

Here we simply check that all of the parameters we set above actually return
the correct information.

```
  context 'socks' do
    it 'should accept a name' do
      socks[:name].should eq 'test'
    end
    it 'should accept a local_port' do
      socks[:local_port].should eq '8080'
    end
    it 'should accept a socks boolean' do
      socks[:socks].should eq :true
    end
    it 'should accept a remote_server' do
      socks[:remote_server].should eq 'testserver'
    end
  end



  context 'tunnel' do
    it 'should accept a name' do
      tunnel[:name].should eq 'test2'
    end
    it 'should accept a local_port' do
      tunnel[:local_port].should eq '8080'
    end
    it 'should accept a socks boolean' do
      tunnel[:socks].should eq :false
    end
    it 'should accept a remote_server' do
      tunnel[:remote_server].should eq 'testserver'
    end
    it 'should accept a forward_server' do
      tunnel[:forward_server].should eq 'localhost'
    end
    it 'should accept a forward_port' do
      tunnel[:forward_port].should eq '80'
    end
  end
```

Now we test our validations.  We create various types without parameters.  We
rely on rspecs expect{} functionality.  It allows us to say “we expect{ a thing
}.to DoSomething”, and in our case we expect it to raise an error, specifically
a Puppet::ResourceError with the string we test for.

```
  context 'validation' do
    it 'should raise an error without forward_server' do
      expect { Puppet::Type.type(:ssh_tunnel).new(
        :name           => 'test2',
        :forward_port   => '80',
        :local_port     => '8080',
        :socks          => :false,
        :remote_server  => 'testserver')
      }.to raise_error(Puppet::ResourceError, 'Validation of Ssh_tunnel[test2] failed: forward_server is not set')
    end
    it 'should raise an error without forward_port' do
      expect { Puppet::Type.type(:ssh_tunnel).new(
        :name           => 'test2',
        :forward_server => 'localhost',
        :local_port     => '8080',
        :socks          => :false,
        :remote_server  => 'testserver')
      }.to raise_error(Puppet::ResourceError, 'Validation of Ssh_tunnel[test2] failed: forward_port is not set')
    end
    it 'should fail with a non-boolean socks' do
      expect { Puppet::Type.type(:ssh_tunnel).new(
        :name           => 'test2',
        :local_port     => '8080',
        :socks          => :no,
        :remote_server  => 'testserver')
      }.to raise_error(Puppet::ResourceError, 'Parameter socks failed on Ssh_tunnel[test2]: Invalid value :no. Valid values are true, false. ')
    end
    it 'should fail with a local_port out of range' do
      expect { Puppet::Type.type(:ssh_tunnel).new(
        :name           => 'test2',
        :local_port     => '99999',
        :socks          => :false,
        :remote_server  => 'testserver')
      }.to raise_error(Puppet::ResourceError, 'Parameter local_port failed on Ssh_tunnel[test2]: Port is not in the range 1-65535')
    end
  end

end
```

##Provider

Providers are slightly more difficult to test.  It can help to create the it{}
blocks up front and then fill in the things that fail until the tests pass,
when retrofitting.  We’ll refer to the provider we’re testing by ‘subject’,
which is the “thing between describe and do”, and expect two existing tunnels
running that we’ll stub out later.  In our example this might look like.

```
    it do
      instances = subject.class.instances.map { |p| {:name => p.get(:name), :ensure => p.get(:ensure)} }
      instances[0].should == {:name => '8080:localhost:80-tunnel@remote', :ensure => :present}
      instances[1].should == {:name => '8080-socks@remote', :ensure => :present}
    end

```

That’s a pretty confusing looking it block, so I’ll try to describe what we’re
doing.  First we call ‘subject.class.instances’ which will get the instances
that we stub out later.  We then run .map on the array of instances and pull
out the :name and :ensure status from each one.  This is all stuffed into
instances as an array of hashes.

We then check each hash independently via instances[0] and instances[1] to look
for the name and ensure status we expect to be present.  This, then, would test
that self.instances is able to run and return an appropriate list of instances
given fake data.

Working from the top of the file again, we fill in two type and provider
instances as well as stub out the returns from ssh_processes:

```
require 'spec_helper'

describe Puppet::Type.type(:ssh_tunnel).provider(:ssh) do
  let(:socks_resource) {
    Puppet::Type.type(:ssh_tunnel).new(
      :name          => '8080-socks@remote',
      :socks         => :true,
      :remote_server => 'socks@remote',
      :local_port    => '8080',
      :provider       => :ssh
    )
  }
  let(:tunnel_resource) {
    Puppet::Type.type(:ssh_tunnel).new(
      :name           => '8080:localhost:80-tunnel@remote',
      :socks          => :false,
      :forward_server => 'localhost',
      :forward_port   => '80',
      :remote_server  => 'tunnel@remote',
      :local_port     => '8080',
      :provider       => :ssh
    )
  }
  let(:socks_provider) { socks_resource.provider }
  let(:tunnel_provider) { tunnel_resource.provider }

  describe 'self.instances' do
    before :each do
      subject.class.expects(:ssh_processes).with('-fN -D').returns({'8080 socks@remote' => '10'})
      subject.class.expects(:ssh_processes).with('-fN -L').returns({'8080:localhost:80 tunnel@remote' => '11'})
    end

    it do
      instances = subject.class.instances.map { |p| {:name => p.get(:name), :ensure => p.get(:ensure)} }
      instances[0].should == {:name => '8080:localhost:80-tunnel@remote', :ensure => :present}
      instances[1].should == {:name => '8080-socks@remote', :ensure => :present}
    end
  end
```

Now we have a very short prefetch test that just makes sure it runs without
errors:

```
  describe 'self.prefetch' do
    it 'exists' do
      socks_provider.class.instances
      socks_provider.class.prefetch({})
    end
  end
```

Our next real test is our ‘create’ test.  We ‘expect’ the required arguments
that would be sent to ssh() for a successful tunnel to be created, stub out the
exists?() method return of true, and then run create() and ensure we get no
errors:

```
  describe 'create' do
    it 'makes a socks proxy' do
      socks_provider.expects(:ssh).with('-fN', '-D', '8080', 'socks@remote').returns(true)
      socks_provider.expects(:exists?).returns(true)
      socks_provider.create.should be_true
    end
    it 'makes a tunnel' do
      tunnel_provider.expects(:ssh).with('-fN', '-L', '8080:localhost:80', 'tunnel@remote').returns(true)
      tunnel_provider.expects(:exists?).returns(true)
      tunnel_provider.create.should be_true
    end
  end
```

Our destroy test sets the pid for each provider instance first, then stubs out
the kill we sent to it as succeeding:

```
  describe 'destroy' do
    it 'destroys socks' do
      socks_provider.pid = 10
      Process.stubs(:kill).with('SIGTERM', 10).returns(true)
      socks_provider.destroy.should be_true
    end
    it 'destroys tunnel' do
      tunnel_provider.pid = 11
      Process.stubs(:kill).with('SIGTERM', 11).returns(true)
      tunnel_provider.destroy.should be_true
    end
  end
```

For exists? we simply set the ensure parameter to :present, then call exists?()
and assert that it should return true:

```
  describe 'exists?' do
    it 'checks if socks proxy exists' do
      socks_provider.ensure= :present
      socks_provider.exists?.should be_true
    end
    it 'checks if tunnel proxy exists' do
      tunnel_provider.ensure= :present
      tunnel_provider.exists?.should be_true
    end
  end
```
