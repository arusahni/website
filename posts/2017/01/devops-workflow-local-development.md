<!--
    .. title: A DevOps Workflow, Part 1: Local Development
    .. slug: devops-workflow-local-development
    .. date: 2017-01-09 09:48:21 UTC-05:00
    .. tags: tech, devops
    .. link:
    .. description: The first part of a series on my sample devops process, starting at local development.
-->

_This series is a longform version of an internal talk I gave at a former
company. It wasn't recorded. It has been mirrored here for posterity._

How many times have you heard: "That's weird - it works on my machine?"

How often has a new employee's first task turned into a days-long effort,
roping in several developers and revealing a surprising number of undocumented
requirements, broken links and nondeterministic operations?

How often has a release gone south due to stovepiped knowledge, missing
dependencies, and poor documentation?

In my experience, if you put a dollar into a swear jar whenever one of the
above happened, plenty of people would be retiring early to spend time on their
private islands. The fact that this situation exists is a huge problem.

What would an ideal solution look like? It should ensure consistency of
environments, capture external dependencies, manage configuration, be
self-documenting, allow for rapid iteration, and be as automated as possible.
These features - the intersection of development and operations - make up the
practice of DevOps. The solution shouldn't suck for your team - you need to
maximize buy-in, and that can't be done when people need to fight container
daemons and provisioning scripts every time they rebase to master.

In this series, I'll be walking through how we do DevOps at HumanGeo. Our
strategy consists of three phases - local development, continuous integration,
and deployment.

Please note that, while I mention specific technologies, I'm not stating that
this is The One True Way™. We encourage our teams to experiment with new tools
and methods, so this series presents a model that several teams have
implemented with success, not official developer guidelines.

<!-- TEASER_END -->

# Development Environment: Vagrant

In order to best capture external dependencies, one should start with a blank
slate. Thankfully, this doesn't mean a developer has to format her computer
each time she takes on a new project. Depending on the project, it may be as
simple as putting code into a new directory or creating a new virtual
environment. However, given the scale of the problems we tackle at HumanGeo, we
need to push even further and assemble specific combinations of databases,
Elasticsearch nodes, Hadoop clusters, and other bespoke installations. To do
so, we need to create sandboxed instances of the aforementioned tools; it's the
only sane way to juggle multiple versions of a product when developing locally.
There are plenty of fine solutions to this problem, Docker and Vagrant being
two of the major players. There's not a perfect overlap between the two, but as
they fit in our stack, they're near-equivalent. Since it provides a gentler
learning curve, this series will cover Vagrant.

[Vagrant](http://vagrantup.com/) provides a means for creating and managing
portable development environments. Typically, these reside in
[VirtualBox](https://www.virtualbox.org/) virtual machines, although
they have support for many different backend providers. What's neat is that,
with a single `Vagrantfile`, you can provision and connect multiple VMs, while
automatically syncing code changes made on the host machine (i.e., your
computer) to the guest instance (i.e., the Vagrant box).

To get started with Vagrant, you must define your configuration in a
`Vagrantfile`. Here's a sample:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "trusty64"
  config.vm.hostname = "webserver"
  config.vm.network :private_network, ip: "192.168.0.42"

  config.vm.provider :virtualbox do |vb|
    vb.customize [
      "modifyvm", :id,
      "--memory", "256",
    ]
  end

  config.vm.provision :shell, path: "bootstrap.sh"
end
```

This defines an Ubuntu 14.04 (Trusty Tahr) machine with a fixed private IP,
256mb of RAM, and a bootstrap shell script, which will install needed
dependencies and apply software-level configuration. The `Vagrantfile` can be
committed to version control alongside the bootstrap script and your
application code so the entire environment can be captured in a single
snapshot.

Launching the machine is done with a single command: `vagrant up`. Vagrant will
download the `trusty64` base image from a [central
repository](https://atlas.hashicorp.com/boxes/search), launch a new instance of
it with the hardware and networking states we've defined, and then run the
bootstrap file. The image download will only occur once-per image, so
future machine initializations will utilize the cached version. Machines can be
stopped with `vagrant down`. You can later re-launch the machine with `vagrant
up`.  If you decide that you need to nuke your entire environment from orbit
and start over (an immensely useful option), you can do so with `vagrant
destroy`.

To manage these machines, one can connect via SSH just as one would a remote
server. The `vagrant ssh` command will automatically log the user in using public
key authentication. From there, a developer can experiment with configuration
and other aspects of application development. All ports are exposed to the host
machine, so, if a webserver is bound to port 5000, it can be reached from your
browser at `http://192.168.0.42:5000` (the IP address we assigned to our instance
in the `Vagrantfile`).

Unlike when working with a remote server, you don't need to run a
terminal-based editor via SSH, or use rsync every time you save a file in order
to make changes to the code on the virtual machine. Instead, the directory that
contains your `Vagrantfile` is automatically mounted as `/vagrant/` on the guest,
with changes automatically synced back and forth. So, you can use whatever
editor you want on the host, while executing code on the VM. Easy.

# Provisioning: Ansible

Vagrant itself is only really focused on the orchestration of virtual machines;
the configuration of the machines is outside of its purview. As such, it relies
on a provisioner - an external tool or script that runs against newly created
virtual machines in order to build upon the base image. For example, a
provisioner would be responsible for taking a blank Ubuntu installation and
installing PostgreSQL, initializing a database, and seeding the database with
data.

The example `Vagrantfile` uses a simple shell script (`bootstrap.sh`) to handle
provisioning. For simple cases, this may well be sufficient. However, if you're
doing any serious development, you'll want to move to a more robust
configuration management tool. Vagrant ships with support for several different
ones, including our preferred tool - Ansible.

[Ansible](http://ansible.com/) is great in many ways: its YAML-based
configuration language is clean and logical, it operates over SSH, has a great
community, emphasizes modularity, and doesn't require any custom software be
present on your target computers (other than Python 2, with Python 3 support in
the technical preview phase). With a little elbow grease, you can even make it
idempotent, so there's nothing to fear if you reprovision an instance. Since
these provisioning scripts live alongside your code, they can be included in
your merge review process, and improve validation of your infrastructure.

Swapping out Vagrant's shell provisioner is extremely straightforward. Just
change your provisioner to "ansible", point it at the Ansible configuration
script (called a playbook), and you're set! The final provisioning block should
now look like this:

```ruby
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "playbook.yml"
end
```

## Tasks

Ansible's basic building block is a task. Conceptually, a task is an atomic
operation. These operations run the gamut from the basic (e.g., set the
permissions on a file) to the complex (e.g., create a database table). Here's a
sample task:

```yaml
- name: Install database
  apt: name=mysql-server state=present
```

The equivalent shell command would be `sudo apt-get install mysql-server`.
Nothing fancy, right?

```yaml
- name: Deploy DB config
  copy: src=mysql.{{env_name}}.conf dest=/etc/mysql.conf mode=644
```

There are several things going on here. First, surprise! Ansible is awesome and
speaks [Jinja2](http://jinja.pocoo.org/). As such, it will interpolate the
variable `env_name` into the string value for `src`, resulting in
`mysql.dev.conf` if we were targeting a dev environment (`env_name` is a
convention we use internally for this very purpose).  Next, we're invoking the
[copy module](http://docs.ansible.com/ansible/copy_module.html). This doesn't
actually copy a file from one remote location to another, it instead copies a
local file to a remote destination. This saves you from having to scp the file
to your target machine, then remote in to set a permission. It's also far
easier to understand at a glance.

```yaml
- name: Start mysqld
  service: name=mysql state=started enabled=yes
```

Finally, we ensure that the MySQL service is not only running, but is set to
automatically start when the system does. This highlights one of the benefits
of Ansible's module system - it masks (and handles) underlying implementation
complexities. Whether or not the target machine is using SysV-style inits,
Upstart, or systemd, the service module takes care of it for you.

## Roles

Tasks can either reside in your playbook, or they can be organized into
functional units called roles. Roles not only allow you to group tasks, but
also bundle files, templates and other resources, providing for a clean
separation of concerns. The tasks above can be placed in a file called
`tasks/main.yml`, resulting in the following directory structure:

```
roles
└── mysql                   # Tasks to be carried out on DB machines
    ├── files
    │   ├── mysql.dev.conf
    │   └── mysql.prod.conf
    └── tasks
        └── main.yml
```

Then, all you need to do is reference the role from within your playbook.

## Playbooks

These are the entry points for Ansible. A playbook is comprised of one or more
plays, each of which possesses several parameters: one or more instances to
target, variables to bundle, and a series of tasks (or roles) to execute.

```yaml
- name: Configure the test environment app server
  hosts: 192.168.1.1
  vars:
    env_name: dev
    es_version: 2.1.0
  roles:
    - common
    - elasticsearch
    - mysql
```

What is evident in the above example is how Ansible roles help improve
modularity and reusability. If I have to install MySQL on several different
hosts (e.g., the test app server and the production app server), all I need to
do is include the role. Ansible maintains a [central
repository](https://galaxy.ansible.com/) of roles for developers to customize;
most of the time you don't need to write any novel provisioning code.

To invoke the playbook, run `ansible-playbook name-of-playbook.yml`. If you're
using Ansible with Vagrant, you should instead use `vagrant provision`, as
Vagrant will handle the mapping of hosts and authentication. And, no matter how
many times you provision, the machine state should remain the same.

_This concludes the local development portion of our dive into DevOps. In the
next installment, we'll cover [continuous
integration](link://slug/devops-workflow-ci)!_
