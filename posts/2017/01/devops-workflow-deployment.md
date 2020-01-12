<!--
    .. title: A DevOps Workflow, Part 3: Deployment
    .. slug: devops-workflow-deployment
    .. date: 2017-01-26 09:48:21 UTC-05:00
    .. tags: tech, devops
    .. link:
    .. description: The third and final part of a series on my sample devops process, focusing on deployment.
-->

_This series is a longform version of an internal talk I gave at a former
company. It wasn't recorded. It has been mirrored here for posterity._

Congratulations, [your code looks good](link://slug/devops-workflow-ci)! Now
all you need to do is put your application in front of your users to discover
all the creative ways they'll break it. In order to do this, we'll have to
create our instances, configure them, and deploy our code.

# CloudFormation: Infrastructure Definition

[Amazon Web Services (AWS)](https://aws.amazon.com/) is a common target for
HumanGeo deployments. Traditionally, when one creates resources on AWS, one
uses [the management console interface](https://aws.amazon.com/console/). While
this is a good way to experiment with an environment, it cannot be automated,
nor can it be managed under version control. Amazon, recognizing that the web
console is insufficient for serious provisioning and scaling purposes, provides
a series of tools for application deployment. The one that best fits our needs
is [CloudFormation](https://aws.amazon.com/cloudformation/).

CloudFormation allows you to define your infrastructure as a collection of JSON
objects. For example, an [EC2](https://aws.amazon.com/ec2/) instance can be
declared with the following block:

```json
"ElasticSearchInstance": {
    "Properties": {
        "EbsOptimized": "true",
        "ImageId": { "Ref": "ImageId" },
        "InstanceType": { "Ref": "EsInstanceType" },
        "KeyName": { "Ref": "KeyName" },
        "NetworkInterfaces": [{
            "DeviceIndex": "0",
            "GroupSet": [
                { "Ref": "ElasticsearchSecurityGroup" },
                { "Ref": "SSHSecurityGroup" }
            ],
            "PrivateIpAddress": "10.0.0.11",
            "SubnetId": { "Ref": "Subnet" }
        }],
        "Tags": [{
            "Key": "Application",
            "Value": { "Ref": "AWS::StackId" }
        }, {
            "Key": "Class",
            "Value": "project-es"
        }, {
            "Key": "Name",
            "Value": "project-es01"
        }]
    },
    "Type": "AWS::EC2::Instance"
}
```

If you're familiar with EC2, much of the above should make sense to you. Fields
with `Ref` objects are cross-references to other resources in the CloudFormation
stack - both siblings and parameters. Once written, the JSON document can be
uploaded to AWS and then run. What's really cool here is that we can do this
with an Ansible task!

Since we prefer to maintain a separation between our instance provisioning and
cloud provisioning scripts, our CloudFormation tasks usually reside in a
standalone playbook named `amazon.yml`.

```yaml
- name: Apply the CloudFormation template
  cloudformation:
    stack_name: proj_name
    state: present
    region: "us-east-1"
    template: "files/project-cfn.json"
    template_parameters:
      KeyName: project-key
      EsInstanceType: "r3.large"
      ImageId: "ami-d05e75b8"
    tags:
      Stack: "project-core"
```

This will not only upload the stack template to AWS, but it also will
instantiate the stack with the provided parameters, which can be either
constants or Ansible variables. The world is your oyster! Unlike with other AWS
wrappers, CloudFormation is stateful, storing stack identifiers and only
updating what needs to be updated.

After working with CloudFormation at scale, some warts really started getting
to us - many of which stemmed from the fact that the templating language is
JSON. Updating a template is painful. Its usage of strings instead of variables
makes validation difficult, and there can be a significant amount of repetition
if you have several similar resources. Thankfully, there exists a solution in
the form of the awesome Python library
[troposphere](https://github.com/cloudtools/troposphere). It provides a way to
write a CloudFormation template in Python, with all the benefits of a full
programming language. The tropospheric equivalent of our Elasticsearch stack
is:

```python
t = Template()
t.add_resource(Instance(
    'ElasticSearchInstance',
    IamInstanceProfile=Ref(es_iam_instance_profile),
    ImageId=Ref(ami_id),
    InstanceType=Ref(es_instance_type),
    KeyName=Ref(key_name),
    Tags=Tags(Application=Ref(stack_id), Name='project-es01', Class='project-es'),
    NetworkInterfaces=[NetworkInterfaceProperty(
        GroupSet=[Ref(ssh_sg), Ref(es_sg)],
        AssociatePublicIpAddress=True,
        DeviceIndex='0',
        DeleteOnTermination=True,
        SubnetId=Ref(subnet),
        PrivateIpAddress='10.0.0.11',
    )],
    EbsOptimized=False,)))
print(t.to_json())
```

Since we're using bare variable names, we can use static analysis tools like
Pylint to validate the template. Additionally, now everything can be scripted!
Want to make multiple instances with the same configuration? With JSON, you
were stuck copy-pasting the same chunks of text multiple times. With
troposphere, it's just a matter of wrapping the instance definition in a
function and invoking it multiple times.

When you're ready to apply your template, simply execute it to get
CloudFormation-compatible JSON, and you're good to go.

# Provisioning: Ansible

Ansible was discussed in [the local development
post](link://slug/devops-workflow-local-development), but here it is again!
Assuming you were principled in writing your local development playbook, aiming
at the AWS cloud is pretty straightforward.

First, you'll need to make Ansible aware of your cloud instances. Sure, you
could manually define the host IPs in your inventory, but that means you'll
have to manually update the mapping of hosts to IP addresses any time you need
to recreate an instance. If only there was some way to dynamically target these
machines...

![Thinking](https://i.imgur.com/L9i884p.png)

There is! [Ansible provides a means to use a dynamic inventory backed by
AWS](http://docs.ansible.com/ansible/intro_dynamic_inventory.html#example-aws-ec2-external-inventory-script).
Once you have your credentials configured, you can use any set of EC2
attributes to target your resources. Since we tend to provision clusters of
machines in addition to standalone instances, it'd be nice to have a more
general attribute selector than `tag_Name_project_es01`. This can be accomplished
by applying our own ontology to our EC2 instances using tags. Notice the `Class`
tag in the CloudFormation examples above. While every Elasticsearch instance we
deploy will have a different `Name` tag, they'll all share a `Class` tag of
`tag_Class_project_es`. Get in the habit of using the project name as a prefix
everywhere since tags are global for your account.

When using the dynamic inventory, plays look like this:

```yaml
- name: Build Elasticsearch instances
  hosts: tag_Class_project_es
  gather_facts: yes
  remote_user: ubuntu
  become: yes
  become_method: sudo
  roles:
    - common
    - es
```

With that, `ansible-playbook -i inventory/ production.yml --private-key
/path/to/project.pem` will target all EC2 instances with a `Class` tag of
`project-es` and apply the `common` and `es` roles.

One other aspect of your cloud deployment is that it may require secrets. You
may need to store passwords for an emailer or private keys for encrypted
RabbitMQ channels. Under normal circumstances, these wouldn't (and shouldn't)
be stored in version control. However, once again our good pal Ansible swings
by to help us out. Enter, [Ansible
Vault](http://docs.ansible.com/ansible/playbooks_vault.html).

For this example, we want to manage SMTP credentials using Ansible. First,
let's create a secrets file: `ansible-vault create vars/secrets.yml`

You'll be prompted for a password. Remember this, as it's the only way you can
decrypt the file. Now, lets add the variables to our file:

```yaml
smtp_username: smtp_user@mydomain.biz
smtp_password: s3cur3!
```

Save and exit. Ansible Vault will automatically encrypt the contents. Now, you
can reference the secrets in your playbook:

```yaml
- name: Build Elasticsearch instances
  hosts: tag_Class_project_es
  gather_facts: yes
  remote_user: ubuntu
  become: yes
  become_method: sudo
  roles:
    - common
    - es
  vars_files:
    - vars/secrets.yml
```

Your roles don't need to know that the variables are encrypted, you can
reference them just as you would any other variable. Decryption happens at
runtime, and requires an additional argument be provided: `ansible-playbook -i
inventory/ production.yml --private-key /path/to/project.pem --ask-vault-pass`.
When run, Ansible will prompt you for the vault password. If it's correct, the
file will be decrypted and the variables will be available for your tasks. If
the password is incorrect, you're dropped back to the console and prompted to
try again.

# Continuous Delivery: Jenkins

At this point, we're now able to fully provision our cloud deployments with
Ansible. Most teams stop here, but we take it even further. It's immensely
useful to developers and other stakeholders to see just how things are shaping
up, and catch any undetected integration bugs. To achieve this, we turn once
more to our friends, Ansible and Jenkins, to create a test environment for us.

First, in order to prevent bugs in test from affecting the production
environment, we must instantiate a standalone test environment. It's up to you
to determine just how substantial this abstraction will be. For our purposes,
we'll be creating separate EC2 instances, but nothing else. This is where
troposphere once more proves its worth. We can wrap the generation of the
relevant stack components behind a function that returns either a set of "prod"
resources, or "test" ones. For example:

```python
def get_instance(*, is_production):
    suffix = '-test' if not is_production else ''
    return Instance(
        f'MyInstance{suffix}',
        # edited for brevity
        Tags=Tags(Application=Ref(stack_id),
                  Name=f'project-instance{suffix}',
                  Class=f'project-instance{suffix}'),
        NetworkInterfaces=[NetworkInterfaceProperty(
            PrivateIpAddress=f'10.0.0.{1 + (10 if is_production else 100)}',
        )],)))

t.add_resource(get_instance(is_production=True))
t.add_resource(get_instance(is_production=False))
```

Once those resources are instantiated, it's a simple matter of tweaking your
production playbook to support test-environment specific features - e.g.,
enable verbose logging and debug mode.

This gets you most of the way to a fully automated test environment. However,
there is the small matter of what actually triggers the Ansible deployment.
Jenkins comes to the rescue… again. Since we're deploying from our main develop
branch, we can set up a downstream project that provisions the test instance as
follows:

1. Install the [Ansible plugin for
   Jenkins](https://wiki.jenkins-ci.org/display/JENKINS/Ansible+Plugin)
2. Create a new project.  
    ![Project creation prompt](https://i.imgur.com/PrvF7CY.png)
3. Have it run only when the project that periodically validates our central
   branch succeeds.  
    ![Project trigger prompt](https://i.imgur.com/VQtjiD9.png)
4. When this project runs, it should invoke the Ansible playbook.  
    ![Ansible playbook prompt](https://i.imgur.com/Iq3bPBf.png)

You will have to make your private key available to Jenkins. If this concerns
you, you can also provision an additional set of private keys solely for
Jenkins, so, if you need to revoke access in the future, you don't have to go
through the hassle of creating new private keys for the main account.

Now, when someone pushes to your development branch and the code is
satisfactory (i.e., passing unit tests and lint checks), Jenkins will update
the contents of your test server. Pure developer-driven architecture!

You can take this one step further and have Jenkins do something similar for
your production environment, just triggered differently. This could utilize a
[GitFlow](https://github.com/nvie/gitflow)-like branching strategy where the
master branch contains production-quality code, so updates there trigger a
production deployment.  Jenkins is pretty flexible, so, more often than not,
you can contrive a combination of triggers and preconditions that will
downselect to the conditions that you want to cause a deployment.

# Monitoring

Congrats! Now that you've got a fully automated and tested workflow, what are
you going to do?

![Disneyworld](https://i.imgur.com/IdJy6vg.png)

Nope. You've got this masterpiece, yes, but how do you know it's actually
running? When one manually deploys code, one usually follows up by clicking
around on the site, checking out system load, etc. Jenkins does nothing of the
sort, and neither does your application. Without manually checking, how do you
know that the last commit didn't have an overly verbose method that's spammed
your system logs to the point you don't have any free space? Or, how do you
know that [Supervisord](http://supervisord.org/) wasn't misconfigured and is
actually stopped? Much like with Schrödinger's cat, you don't. Time to start
monitoring our stack.

Thankfully, commercial and free monitoring solutions can do this for you
([except for Nagios - please
stop](http://www.slideshare.net/superdupersheep/stop-using-nagios-so-it-can-die-peacefully)).
If your disk is full, send an email to the ops team. If you're starting to see
some slow queries manifest themselves, nag the nerds in your #developers Slack
channel. The tool we've moved to from Nagios is [Sensu](https://sensuapp.org/).

Sensu utilizes a client-server model, where the central server coordinates
checks across a variety of nodes, and nodes run a client service that executes
checks and collects metrics. Each client-side heartbeat sends data to the
central server, which manages alerting states, etc. We also run
[Uchiwa](http://uchiwa.io/), which provides a great dashboard for managing your
monitoring environment.

Internally, we run a single Sensu server, to which all of our clients connect
via an encrypted RabbitMQ channel. The encryption keys (and configuration) are
deployed to the clients via a common `sensu` Ansible Vault-protected role. What
varies from deployment to deployment are the checks that are carried out.
Within our playbooks, we define various subscriptions for a set of machines…

```yaml
- name: Monitor the collector instance
  hosts: tag_Class_project_collector
  remote_user: ubuntu
  become: yes
  become_method: sudo
  roles:
    - role: sensu
      subscriptions: [ disk, collector, elasticsearch ]
```

We then customize the block of tasks that provision our checks, using `when`
clauses that test the `subscriptions` collection:

```yaml
- name: check elasticsearch cluster health
  sensu_check:
    name: elastic-health
    command: "{{ plugins_base_path }}/check-es-cluster-status.rb -h {{ hostvars[groups['tag_Name_project_es01'][0]]['ec2_private_ip_address'] }}"
    handlers: project-mailer
    standalone: yes
    interval: 300
  notify: restart sensu-client service
  when: "'elasticsearch' in subscriptions"
```

There are two awesome things going on here. The first is that Ansible ships
with a Sensu check module. This is far nicer than maintaining our own templated
JSON. Additionally, check out that `hostvars` statement! This taps into the power
of the AWS dynamic inventory to allow for attribute lookups.

# That's It

Whew! After all this, what does our final provisioning setup look like?

```
.
├── amazon.yml                 # Provision our AWS infrastructure
├── files                      # Top-level provisioning resources
│   ├── Makefile               # Compiles the troposphere script
│   ├── project-cfn.py         # troposphere script
│   ├── project-stack.json     # Compiled troposphere script
├── group_vars
│   └── all
│       └── vars.yml           # Global Ansible vars
├── inventory
│   ├── base
│   ├── ec2.ini
│   └── ec2.py                 # Ansible AWS discovery script
├── production-monitoring.yml  # Provision infrastructure with Sensu software
├── production.yml             # Provision infrastructure with software
└── roles
    ├── cloudformation         # Applies the CloudFormation template with our parameters
    │   └── tasks
    │       └── main.yml
    ├── es
    │   ├── defaults
    │   │   └── main.yml       # Default ES variables, can be overridden by plays
    │   ├── tasks
    │   │   ├── main.yml       # ES provisioning tasks
    │   └── templates
    │       ├── elasticsearch.yml.j2 # Jinja2 ES config template
    │       ├── kibana.yml.j2  # Kibana config template
    │       └── nginx.conf.j2  # NGINX config template
    └── sensu
        ├── defaults
        │   └── main.yml       # Ansible Vault encrypted Sensu certificates and passwords
        ├── files
        │   └── sudoers-sensu  # A config file to be applied on some systems. Not templated.
        ├── handlers
        │   └── main.yml       # Tasks that are triggered by changes within Sensu
        ├── tasks
        │   ├── checks.yml     # Tasks that are activated on each target based on subscriptions
        │   └── main.yml       # Baseline Sensu client installation
        ├── templates
        │   ├── client.json.j2      # Local Sensu client configutation template
        │   └── rabbitmq.json.j2    # RabbitMQ configuration to communicate
        └── vars
            ├── main.yml            # Base Sensu client configuration
            └── sensu_rabbitmq.yml  # RabbitMQ configuration
```

While what's been presented in this series may seem imposing, it really isn't
if you take it one step at a time. Everything builds off the previous work, and
even if you only implement a subset of the solution presented, you still get to
reap the rewards of a better-managed, more consistent stack.
