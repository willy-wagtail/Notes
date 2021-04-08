# Ansible notes

1. [Inventory](#1-inventory)
2. [Connection settings and privilege escalation](#2-connection-settings-and-privilege-escalation)
3. [Adhoc commands](#3-adhoc-commands)

## 1 Inventory

- An inventory defines a collection of hosts managed by Ansible
- Hosts can be assigned to groups
- Groups can be managed collectively
- Groups can contain child gruops
- hosts can be members of multiple groups
- vars can be set that apply to hosts and gruops

Define using text file
- can be written in INI-style or in .yml
- this is called _static inventory_ - needs to be manually updated
- a _dynamic inventory_ is auto generated and updated using scripts

Inventory location is controlled by your current Ansible configuration file
- ``ansible --version`` will show you which configuration file is in use
- location of inventory is defined in that config file like this:
 ```
 [defaults]
 inventory = ./inventory
 ```
- if this isnt set, ``/etc/ansible/hosts`` is used by default

INI-formatted inventory file - simplest form
- list of host names or IP addresses, e.g.
  ```
  web1.example.com
  192.0.2.42
  ```
- host groups allow you to collectively automate a set of systems using square brackets, e.g
  - group names should not include dashes, but underscore is fine
  - avoid confusion - do not give a gruop the same name as a host
  ```
  [webservers]
  web1.example.com
  web2.example.com
  192.0.2.42

  [db_servers]
  db1.example.com
  db2.example.com
  ```
- a host can be a member of multiple groups
- this allows you to organise groups in different ways dependeing on how you want to manage them
  - web services or db servers
  - servers in prod or test
  - servers in different geolocations
  - e.g.
  ```
  [webservers]
  web1.example.com
  web2.example.com
  192.0.2.42

  [db_servers]
  db1.example.com
  db2.example.com

  [east_datacenter]
  web1.example.com
  db1.example.com

  [west_datacenter]
  web2.example.com
  db2.example.com

  [production]
  web1.example.com
  web2.example.com
  db1.example.com
  db2.example.com

  [development]
  192.0.2.42
  ```

Two host groups always exist
- ``all`` includes every host in the inventory
- ``ungrouped`` includes every host in all that is not a memeber of another group

Groups can include other groups called _nested groups_
- in INI-formatted inventory, you can add nested  groups with the ``:children`` suffix
- e.g. usa and canada are inside the north_america group
  ```
  [usa]
  washington01.example.com
  washington02.example.com

  [canada]
  ontario01.example.com
  ontario02.example.com

  [north_america:children]
  canada
  usa
  ```

Range shortcut
- ``192.168.[4:7].[0:255]``
  - matches all IPv4 adddresses in the range ``192.168.4.0`` through ``192.168.7.255`` (the 192.168.4.1/22 CIDR-notation network)
- ``server[01:20].example.com`` 
  - matches all hosts named ``server01.example.com`` through to ``server20.example.com``
- ``[a:c].dns.example.com`` 
  - matches hosts named ``a.dns.example.com``, ``b.dns.example.com`` and ``c.dns.example.com``
- if leading zeros are included in the range, they are used in the pattern.
  - so ontario01.example.com matches, but ontario1.example.com does not
  ```
  [usa]
  washington[1:2].example.com

  [canada]
  ontario[01:02].example.com
  ```

YAML format
```
all:
  children:
    north_america:
      children:
        canada:
          hosts:
            ontario01.example.com: {}
            ontario02.example.com: {}
        usa:
          hosts:
            washington1.example.com: {}
            washington2.example.com: {}
```

``ansible-inventory`` command used to verify inventory
- ``-i`` option can be used to check any file rather than the current inventory

``ansible-inventory -y --list`` will display the current inventory in YAML format
- easy way to convert INI files to YAML files

``ansible washington1.example.com --list-hosts`` - used to verify a machine's presence in the inventory
- outputs ``hosts (1): washington1.example.com``


In this course, ansible control host running RHEL8, managing 2 web servers and two db servers.
- ``vim /etc/ansible/ansible.cfg`` - default ansible config file
  - you can see that the default inventory is configured in ``/etc/ansible/hosts`` file
  - ``vim /etc/ansible/hosts``
- ``vim inventory`` create own inventory file with basic entries for the 4 hosts
  ```
  [webservers]
  web01
  web02

  [databases]
  db[01:02]
  ```
- ``ansible-inventory -i inventory --list`` to list all hosts listed in ``inventory`` file in json format

## 2 Connection settings and privilege escalation

Ansible does not require you to install an agent on managed hosts.

Protocols and software included with the OS are used: 
- ssh and python on linux systems
- other protocols for windows such as Windows remote management and powershell

Advantages of using common well tested and understool tools.
- simpler to prepare systems 
- reduces security risks

Ansible on the control node needs some information to successfully connect to managed hosts.
- the location of the inventory file
- the connection protocol to use (default ssh)
- whether a non-standard network port is needed to connect to the server
- what user it can login as
- if the user is not root, whether ansible should escalate privileges to root
- how ansible should become root (by default with ``sudo``)
- whether to prompt for an ssh password to log in or a ``sudo`` password to gain privileges

You can set default selections for this information in your Ansible configuration file, or passing flags on the command line during invocation.

Ansible chooses its config from one of several locations, in order the given order
- if ``ANSIBLE_CONFIG`` is set, its value is the path to the file
- if environment variable is not set, Ansible will look for config in the following places
  - ``./ansible.cfg`` in the current directory where you ran the ansible command
  - ``~/.ansible.cfg`` as a dot file in the user's home directory
  - ``/etc/ansible/ansible.cfg`` the default config

``ansible --version`` clearly identifies which config file is currently being used
- ``ansible-config --version`` can also get this info

``ansible.cfg`` file
- each section contains settings as key value pairs
- section titles are closed in square brackets
- basic operations use two sections:
  - ``[defaults]`` sets defaults for Ansible operation
  - ``[privilege_escalation]`` configures how Ansible performs privilege escalation on managed hosts

Connect settings in config file is in ``[defaults]`` section
- ``remote_user`` specified the user you want to use on the managed host
  - if none specified, it uses your current user name
- ``remote_port`` specifies which port sshd is using on the managed host
  - if none specified, the default is port 22
- ``ask_pass`` controls whether Ansible will prompt you for the ssh password
  - by default it will NOT prompt for password, assuming you are using SSH key based auth

In ``[privilege_escalation]`` section of config file
- ``become`` controls whether you will auto use privilege escalation
  - default is no, and can override this at the command line or in platbooks
- ``become_user`` controls what user on the managed host Ansible should become
  - default is root
- ``become_method`` controls how Ansible will become that user
  - default uses ``sudo``, there are other options like ``su``
- ``become_ask_pass`` controls whether to prompt you for a password for your become method
  - default is no

The following is a typical ``ansible.cfg`` file
```
[defaults]
inventory = /.inventory
remote_user = ansible
ask_pass = false

[privilege_escalation]
become = true
become_user = root
become_ask_pass = false
```

Host-based connection and privilege escalation variables variables

You can apply settings specific to a particular host by setting connection variables
- easiest is placing the settings in a file in the ``host_vars`` directory in the same directory as the inventory file
  - e.g. see below file directory representation. there are two host-specific files for server1.example.com and server2.example.com
  - the hosts should appear with those names in the inventory
    ```
    project
      ansible.cfg
      host_vars
        server1.example.com
        server2.example.com
      inventory
    ```
- these settings override the ones in ``ansible.cfg``
- they have slightly different syntax and naming

- ``ansible_host`` specifies a diff IP or hostname to use for connection to this host instead of one in inventory
- ``ansible_port`` specifies the port to use for the ssh connection on this host
- ``ansible_user`` specifies whether to use privilege escalation
- ``ansible_become`` specifies the user to become on this host
- ``ansible_become_user`` specfies the user to become on this host
- ``ansible_become_method`` specifies the privilege escalation method to use for this host

Example of ``host_vars/server1.example.com``
  ```
  # connection variables for server1.example.com
  ansible_host: 192.0.2.104
  ansible_port: 34102
  ansible_user: root
  ansible_become: false
  ```
- these settings only affect ``server1.example.com`` in the inventory

Preparation of the managed host
- set up ssh key-based authentication to an unprovileged account that can use ``sudo`` to become root without password
- advantage is that you can use an account that only ansible uses and tie that to a particular ssh private key, but still have passwordless authentication
- alternatively, ssh key-based authentication to the unprivileged account, then require the ``sudo`` password for authentication to ``root``
- ansible allows you to select the mix of settings that works best for your security policy and stance

demo
``vim /etc/ansible/ansible.cfg``
  - search for ``become`` to find the area that deals with this
  - you can see defaults in ``[privileged_escalation]`` section
    - become default is true, method is sudo, user is root, and pass is false

``vim ansible.cfg`` create our own custom ``ansible.cfg``
```
[defaults]
inventory = /home/demo/ansible/inventory

[privilege_escalation]
become=True
become_method=sudo
become_user=root
become_ask_pass=False
```

``ansible --version`` to check that ansible will use our custom config file

``mkdir host_vars`` create a ``host_vars`` directory to contain our host connection settings
- we will create one file per host to specify values that supplies values that override defaults
- ``cd host_vars/``
- ``vim db01``
  ```
  # host variables for the db01 system
  ---
  become_ask_pass: True
  ```

``ansible databases --limit db01 -m ping``
- try running ansible on the databases inventory group, limited to db01 system
- error, because i'm located in teh ``host_vars`` directory. need to go up one level
- ``cd ..``
- rerun ``ansible databases --limit db01 -m ping``

``cd host_vars``
- ``vim db01`` change file by adding line 
  ```
  # host variables for the db01 system
  ---
  become_ask_pass: True
  custom_db_port: 1234
  ```
- ``cd ..``
- ``ansible databases --limit db01 -m ping``

``ansible-config dump --only-changed`` to check what we've overridden with our custom ansible.cfg

After creating and populating our hosts with keys, we can actually just directly ssh into our hosts without supplying a password
- ``ssh web01``
- ``logout``

## 3 Adhoc commands

Ad-hoc commands are simple one line operations that run without writing a playbook useful for quick tests and changes. There are limitations to this approach though.
- e.g. to start a service or ensure a line exists in a file

Ansible provides modules, code that can be used to automate particular tasks
- some use of modules include
  - ensure users exist withc ertain settings
  - make sure latest version of software packages are installed
  - deploy a config file to a server
  - enable a network service and make sure it is running
- most modules are idempotent which means they only make changes if a change is needed. They can safely be run multiple times.
- an ad-hoc command runs one module on the specified managed host

The ``ansible host-pattern -m module [-a 'module arguments'] [-i inventory]`` command runs an ad-hoc command
- supply a ``host-pattern`` argument with the managed host to run on
- the ``-m`` option names the module that ansible should run
- the ``-a`` option takes a list of all arguments required by the module
- the ``-i`` option is where the host can be found

One of the simplest ad-hoc commands is ``ansible all -m ping``. Using the ping module, it calls all hosts.
- it does not send an ICMP ping to managed host like the usual understanding of ping
- it checks to see if ansible modules written in Python can run on the managed host
  - the output showing "SUCCESS" against the host means it was able to contact the host and it replied with the answer ``pong``

To override a default configuration setting there are several different options. These options override the configuration in ``ansible.cfg`` configuration file
- ``-k`` or ``--ask-pass`` will prompt for the connection password
- ``-u REMOTE_USER`` overrides the ``remote_user`` setting in ``ansible.cfg``
- ``-b`` option or ``--become`` enables privilege escalation, running operations with ``become: yes``
- ``-K`` of ``--ask-become-pass`` will prompt for the privilege escalation password
- ``--become-method`` will override the default privilege escalation method
  - default is ``sudo``
  - can find valid choices using ``ansible-doc -t become -l``
- ``-i`` or ``--inventory`` is used to override the inventory file
- ``--become-user`` 

Most modules take arguments to control them using the ``-a`` option to pass arguments
- e.g. ``ansible -m user -a 'name=newbie uid=4000 state=present' servera.lab.example.com``
  - using the user module to ensure the user named newbie exists and has a uid of 4000
  - arguments are inside single brackets and are made with key value pairs separated by a space
  - ``state`` concept is the main way you describe the behaviour you wish it to perform
    - e.g. ``state=absent`` will ask it to go ahead and remove the user

``ansible-doc -l`` lists all modules available to us. It is a long list.
- ``ansible-doc -l | grep ping`` to narrow it down to find ping module
- ``ansible-doc ping`` to get full details of the ping module
- bottom of documentation, there are examples of use.

### Demo

Now we have looked up the docs for the ansible module we want to run:
First, ``ansible --version`` to show default ansible config file to see inventory
- ``cat ansible.cfg`` and we see we are overriding the default ``/etc/ansible/hosts`` inventory file to ``/home/demo/ansible/inventory``
- ``cat inventory``
- ```
  [webservers]
  web01
  web02

  [databases]
  db[01:02]
  ```

Next run command ``ansible all -i inventory -m ping`` to ping all hosts in the inventory.

Can supply additional args, and limit hosts.
- E.g. force a password handshake instead of relying on our SSH keys, supply ``-k`` flag to get ansible to prompt for passwords when authenticating.
- ``ansible all --limit web01 -i inventory -k -m ping`` targets all host, but limiting to a single one.

Lets try another module that manages services
- ``ansible-doc -l | grep service`` - find ``service`` which "manages services"
- ``ansible-doc service`` to look at the documentation for just this module
  - controls services on remote hosts. See examples at the end.

Lets try an ad-hoc command to reestart ``sshd`` on one of our targeted hosts.
- ``ansible all -m service -a "state=restarted name=sshd"`` will restart sshd service in each of the hosts

Lets try creating users on our target machines
- ``ansible-doc user`` to check out the module documentation. Note the "=" sign in the docs denotes mandatory fields (name is one of them).
- ``ansible webservers -m user -a "name=test password=secure_password state=present"`` 
  - target webservers group in inventory using the user module to create a user with password as supplied. The state=present arg means only do so if the user is not already there on the machine
- Try logging in to one of these systems as the newly created user.
- ``ansible webservers -m user -a "name=test password=secure_password state=absent"`` will remove the users

``state`` argument tells ansible the end state in which we want the machine to end up in.

### Selecting modules for ad-hoc commands

Finding information about ansible modules
- ``ansibl-doc -l`` command lists all modules installed on a system
- The name and a description of the module are displayed
- Thousands of modules are available: consider piping the output into a ``grep`` to filter the result.
- The same information is available from the [Ansible website](https://docs.ansible.com/ansible/2.8/modules/list_of_all_modules.html)

Some selected modules include:
- file modules
  - ``copy`` module - copies a local file to the manages host
  - ``file`` module - sets permissions and other properties of files
  - ``lineinfile`` module - checks a particular line is or is not in a file
  - ``synchronize`` module - synchronizes content using rsync
- software package modules
  - ``yum`` module - manages packages using yum
  - ``dnf`` module - manages packages using DNF
  - ``gem`` module - manages Ruby gems
- system modules
  - ``firewalld`` module - manage arbitrary ports and services using firewalld
  - ``reboot`` module - reboots a machine
  - ``service`` module - manages services
  - ``user`` - adds, removes, and manages user accounts
- net tools modules
  - ``get_url`` module - downloads files over http, https, or ftp
  - ``nmcli`` module - manages networking
  - ``uri`` module - interacts with web services and communicates with APIs

Managing package using ``package`` module
- ``ansible-doc package`` to find out how the module works
  - ``name`` is the name of package or a list of packages to manage
  - ``state`` specifies whether the package must be present, absent, or the latest version. This is used to remove or update packages
  - E.g. ``ansible all -m package -a 'name=httpd state=present'`` will ensure the ``httpd`` package is installed on al hosts.
  - other modules are also available including ``yum``, ``dnf``, and ``apt`` that work in a similar way but which might support more sophisticated options specific to those managers.
  - ``package`` module aims to be system agnostic

Command Modules
- handful of modules that run commands directly on the managed host
- use these if no other modules are available to do what you need
- these are not idempotent - you must make sure that they are safe to run twice when using them
- ``command`` runs a single command on the remote system
  - cannot access shell env vars or perform shell operations such as redirection and piping.
  - E.g ``ansible mymanagedhosts -m command -a /usr/bin/hostname``
- ``shell`` runs a command on the remote system's shell (redirection and other features work)
  - ``ansible localhost -m command -a set`` (this will fail)
  - ``ansible localhost -m shell -a set`` (this will change the shell and output BASH=/bin/sh)
- ``raw`` simply runs a command with no processing (can be dangerous)
  - this bypasses module subsystem completely, useful when managing systems that cannot have python installed (e.g. a network router).
- in general, aim to use regular modules before resorting to these

Disadvantages of Ad hoc commands
- only call one module at a time
- type the same command again to rerun it - options can get complex
- manual in nature

Better approach is to use Ansible Playbooks.
- run multiple modules with conditionals and other processing
- text files which can be tracked in version control
- can be rerun with a simple command

### Demo

Goal: Add users to a group we create.

- ``ansible-doc -l | grep group | less`` to find a module to manage group creation
- ``ansible-doc group`` - check group docs
  - options we need are name and state (which defaults to present so can omit).
- ``ansible webservers -m group -a "name=test"``
  - this will create a user group called test
  - rerun it to demonstrate idempotency. The second run will report that the group is already present.
- ``ansible webservers -m user -a "name=test group=demo"``
  - this will add user called test to group demo.
  - rerunning it, will show that it is already in state present
- ``ansible webservers -m user -a "name=test group=demo state=absent"``
  - this will remove the user called test in group demo
  - rerunning it, again will report success without doing anything
- ``ansible webservers -m group -a "name=test state=absent"``
  - this will remove the group called test

Goal: Install apache webserver

- ``ansible webservers -m package -a "name=httpd state=present"`` to add
- ``ansible webservers -m package -a "name=httpd state=absent"`` to remove
- ``ansible webservers -m yum -a "name=httpd state=present"`` to add using yum if using redhat linux system
- ``ansible webservers -m yum -a "name=httpd state=absent"`` to remove
