#Introduction

[RESEARCH:  Facter 2 and the changes within.  ]

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

Keep in mind when considering whether or not to write a fact to gather data
that facts are executed on the client prior to catalog compilation on the
master [??? VERIFY! I know this is on the client, but only somewhat sure the
ordering goes agent requests catalog, master requests facts, master sends back
catalog with facts populated'].  If you need to gather information that is
available on the puppet master you should instead be using custom functions.

The recently released Facter 2.0 has dramatically changed the
functionality and ability to write sophisticated facts.  This document
has been written against Facter 2.0 and contains non-backwards
compatible code.

#Background

When generating private keys via OpenSSH, a pair is created with the
filename id_x and id_x.pub.  The X differs by encryption method used,
but generally it’s id_rsa and id_dsa.  Public keys look like:

```
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDVwwxl6dz6Y7karwyS8S+za4qcu99Ra8H8N3cVHanEB+vuigtbhLOSb+bk6NjxFtC/jF+Usf5FM5fGIYd51L7RE9BbzbKiWb9giFnNqhKWclO5CY4sQTyUyYiJTQKLuVtkmiFeArV+jIuthxm6JrdOeFx8lJpcgGlZjlcBGxp27EbZNGWIlAdvW0ZXy0JqS9M/vj71NBBDfkrpyzAPC0aBa9+FmywOH6HXbyeFooHLOw+mfzP87jwDDQ2yXIehDoC1BsLYXD+j+kdnR0CNltJh1PYOFNpbKQpfnPhfdw4Oc0hZ34n+kfBPavKlbwxoVAoisBWWo4c9ZnUoe2OBRHAX comment at the end
```

#Supporting decisions

Puppet is a declarative system, which means we describe the end
configuration state desired for your nodes.  New puppet users often
make the mistake of writing facts to allow them to build complex
logic based on the local state of the agent being managed.  They then
use these facts to determine what actions to take.  An example of this
might be a fact called `apache_installed` and a manifest that says:

```puppet
if $apache_installed {
  include 'something'
}
```

This is a dangerous way to use facts.  Facts should be used to find
out the information needed to enforce a decision that the module
designer has already decided on, rather than being used to make those
decisions.  In the above example you should decide at a higher level
if apache should be installed or not, and then make it so, not query
the agent to find out if it was.

Writing facts to make decisions rather than supporting them causes the
unfortunate side effect of making your catalog nondeterministic. This
means that it is hard to reason about infrastructure because it is
constantly changing based on the local state of the systems.  It's best
to leave facts in a supporting role of helping you enforce policy that
was predetermined based on the role or profile of the agent, not the
local state of the server being managed.

#Logic

Another common mistake made is to move logic into facts, as opposed to
modeling your logic within the manifest.  Facts should, when
possible, simply return information.  This is then consumed within the
manifest to decide the end state.  Internal logic within the fact
which is used to help obtain the information is fine, but you
shouldn't write a fact called `ssh_required` that returns true/false.
Instead you should write a fact called `ssh_installed` that returns
true/false and then write the appropriate logic to determine if ssh is
required within the manifest based on the knowledge that it's
currently installed or not.

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
the `hardwaremodel.rb` fact distributed with Facter.

#Demonstration

We start our fact by requiring etc, a ruby gem, in order to iterate
through the users on the system.  We then iterate through each entry
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
    # Facts must be strings, so return a list of users comma separated.
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
