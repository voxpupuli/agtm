#Introduction

In the [previous section](Types and Providers) we extended the ssh
module we began in the [Classes, Params, and APIs](Same) section to
allow it to create ssh_tunnels.  Now we’re going to extend the module
to add two new facts.  The first is `private_key_users`, a list of
users that have known private keys in their .ssh/ directory.  The
second is `public_key_comments`, a collection of all the comments from
all *.pub files, in case you need a quick way to confirm that a key
exists on a server.

These aren’t the most practical examples, but they serve as
demonstrations for what you can do in facts.

#Background

By default when generating private keys via OpenSSH a pair is created
with the filename id_x and id_x.pub.  The X differs by encryption
method used, but generally it’s id_rsa and id_dsa.  Public keys look
like:

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDVwwxl6dz6Y7karwyS8S+za4qcu99Ra8H8N3cVHanEB+vuigtbhLOSb+bk6NjxFtC/jF+Usf5FM5fGIYd51L7RE9BbzbKiWb9giFnNqhKWclO5CY4sQTyUyYiJTQKLuVtkmiFeArV+jIuthxm6JrdOeFx8lJpcgGlZjlcBGxp27EbZNGWIlAdvW0ZXy0JqS9M/vj71NBBDfkrpyzAPC0aBa9+FmywOH6HXbyeFooHLOw+mfzP87jwDDQ2yXIehDoC1BsLYXD+j+kdnR0CNltJh1PYOFNpbKQpfnPhfdw4Oc0hZ34n+kfBPavKlbwxoVAoisBWWo4c9ZnUoe2OBRHAX comment at the end
```

#Supporting decisions

Puppet is a declarative system, with an emphasis on building an end
state desired for your nodes.  One common mistake of new puppet users
is to write facts that are then used to make decisions that may change
based on local state.  Facts should be used to find out the
information needed to enforce a decision that the module designer has
already decided on, rather than being used to make those decisions.

For an example of this decision making process we can turn to the
puppetlabs-lvm module, where the facts are used to provide supporting
information to lvm and to enhance the ability to act upon declared end
goals.

Writing facts to make decisions rather than supporting them causes
the unfortunate side effect of making your catalog
nondeterministic. This means that it is hard to reason about
infrastructure because it is constantly changing based on the local
state of the systems.

A real world example of this is when a new user creates a fact that
checks for the existence of a file on the local system and then
installs a package based on this knowledge.  It is far better to
decide if the package and file should or shouldn’t exist and then
express that intent in your manifest, forcing the end state of the
machine to your requirements.

#Logic

Another common mistake made is to move logic into facts, as opposed to
modelling your logic within the manifest.  Facts should, where
possible, simply return information.  This is then consumed within the
manifest to decide the end state.  Logic in facts is fine when it’s
used to obtain the information, but you should write something like
`ssh_required` which returns true/false.  Instead you would write a
fact that returned something like `ssh_installed` and then express
your logic further in the manifest.

#Typing

Confusing new users constantly is the fact that all when Puppet
obtains facts from facter they are all “stringified”, removing all
type information from them.  This means that a fact that returns a
boolean of true/false when seen through running `facter` will become
‘true’ and ‘false’ as strings within Puppet.  Yes, this means that a
fact returning false is considered to be true within puppet when
tested as a boolean, because strings always evaluate to true.  Instead
you have to do:

```
if $fact == ‘false’
```

As only strings are allowed this also means you cannot return hashes
or arrays.  As a way around this some facts will return a comma
seperated list, such as the interfaces fact:

```
interfaces => lo0,gif0,stf0,en0,en3,en2,p2p0
```

You can use functions to create real types from this such as:

```
$interface_list = split($interfaces, ‘,’)
```

#Confines

One of the mechanisms facts supports is confines, allowing you to
write:

```
confine :kernel => ‘Linux’
```

But you can also get more sophisticated and pass in a block to
confines, such as:

```
confine { File.exists?(keyfile) }
```

An example of a fact that relies on a confine block can be found in
the `gid.rb` fact distributed with Facter.

##Demonstration

We start our fact by requiring etc, a ruby gem, in order to iterate
through the users on the system. we then iterate through each entry
of the password file.

```
require 'etc'

Facter.add(:private_key_users) do
  setcode do
    users = []
    # This generates an object for each entry in /etc/passwd
    Etc.passwd do |user|
```

We now look for known private keys and if one is found we add the
users name to the list of users.

```
      # Look for known private key names.
      ['id_rsa', 'id_dsa'].each do |file|
        if File.exists?("#{user.dir}/.ssh/#{file}")
          users << user.name
          # Once we've found a key we can skip the rest of the tests
          break
        end
      end
    end
```

Finally, at the end we return the list of users.

```
    # Facts must be strings, so return a list of users comma seperated.
    users.join(',')
  end
end
```

Our second fact is similar to the first fact, except that we extract
public-key comments and return a comma separated list of these.

```
require 'etc'

Facter.add(:public_key_comments) do
  setcode do
    comments = []
    # This generates an object for each entry in /etc/passwd
    Etc.passwd do |user|
      # Generate a list of *.pub files in ~/.ssh/
      keyfiles = Dir["#{user.dir}/.ssh/*.pub"]
      # Check each .pub file in ~/.ssh/ in turn

```

Lastly, we check each keyfile in turn by reading it and taking the
last element as the comment.  This is fairly error prone as it simply
relies on a space separator so any comment with a space fails, but
it’s concise and matches the default style comments generated with
ssh-keygen.

```
        keyfiles.each do |file|
        contents = File.read("#{user.dir}/.ssh/#{file}")
        contents.each_line do |line|
          comments << line.split(' ').last
        end
      end
    end
    # Facts must be strings, so return a comma seperated list.
    comments.uniq.join(',')
  end
end

```

#Conclusion

[TODO], wrap up text.
