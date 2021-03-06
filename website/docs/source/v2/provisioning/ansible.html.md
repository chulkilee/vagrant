---
page_title: "Ansible - Provisioning"
sidebar_current: "provisioning-ansible"
---

# Ansible Provisioner

**Provisioner name: `"ansible"`**

The ansible provisioner allows you to provision the guest using
[Ansible](http://ansible.com) playbooks.

Ansible playbooks are [YAML](http://en.wikipedia.org/wiki/YAML) documents that
comprise the set of steps to be orchestrated on one or more machines. This documentation
page will not go into how to use Ansible or how to write Ansible playbooks, since Ansible
is a complete deployment and configuration management system that is beyond the scope of
a single page of documentation.

<div class="alert alert-warn">
  <p>
    <strong>Warning:</strong> If you're not familiar with Ansible and Vagrant already,
    I recommend starting with the <a href="/v2/provisioning/shell.html">shell
    provisioner</a>. However, if you're comfortable with Vagrant already, Vagrant
    is a great way to learn Ansible.
  </p>
</div>

## Inventory File

When using Ansible, it needs to know on which machines a given playbook should run. It does
this by way of an [inventory](http://docs.ansible.com/intro_inventory.html) file which lists those machines. In the context of Vagrant,
there are two ways to approach working with inventory files.

The first and simplest option is to not provide one to Vagrant at all. Vagrant will generate an
inventory file encompassing all of the virtual machine it manages, and use it for provisioning
machines. The generated inventory file is stored as part of your local Vagrant environment in `.vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory`.

The `ansible.groups` option can be used to pass a hash of group
names and group members to be included in the generated inventory file. Group variables
are intentionally not supported, as this practice is not recommended.
For example:

```
ansible.groups = {
  "group1" => ["machine1"],
  "group2" => ["machine2", "machine3"],
  "all_groups:children" => ["group1", "group2", "group3"]
}
```

Note that unmanaged machines and undefined groups are not added to the inventory.
For example, `group3` in the above example would not be added to the inventory file.

A generated inventory might look like:

```
# Generated by Vagrant

machine1 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2200
machine2 ansible_ssh_host=127.0.0.1 ansible_ssh_port=2201

[group1]
machine1

[group2]
machine2

[all_groups:children]
group1
group2
```

The second option is for situations where you'd like to have more control over the inventory management.
With the `ansible.inventory_path` option, you can reference a specific inventory resource (e.g. a static inventory file, a [dynamic inventory script](http://docs.ansible.com/intro_dynamic_inventory.html) or even [multiple inventories stored in the same directory](http://docs.ansible.com/intro_dynamic_inventory.html#using-multiple-inventory-sources)). Vagrant will then use this inventory information instead of generating it.

A very simple inventory file for use with Vagrant might look like:

```
default ansible_ssh_host=192.168.111.222
```

Where the above IP address is one set in your Vagrantfile:

```
config.vm.network :private_network, ip: "192.168.111.222"
```

Note that machine names in `Vagrantfile` and `ansible.inventory_path` file should correspond,
unless you use `ansible.limit` option to reference the correct machines.

## Playbook

The second component of a successful Ansible provisioner setup is the Ansible playbook
which contains the steps that should be run on the guest. Ansible's
[playbook documentation](http://docs.ansible.com/playbooks.html) goes into great
detail on how to author playbooks, and there are a number of
[best practices](http://docs.ansible.com/playbooks_best_practices.html) that can be applied to use
Ansible's powerful features effectively. A playbook that installs and starts (or restarts
if it was updated) the NTP daemon via YUM looks like:

```
---
- hosts: all
  tasks:
    - name: ensure ntpd is at the latest version
      yum: pkg=ntp state=latest
      notify:
      - restart ntpd
  handlers:
    - name: restart ntpd
      service: name=ntpd state=restarted
```

You can of course target other operating systems that don't have YUM by changing the
playbook tasks. Ansible ships with a number of [modules](http://docs.ansible.com/modules.html)
that make running otherwise tedious tasks dead simple.

## Running Ansible

To run Ansible against your Vagrant guest, the basic Vagrantfile configuration looks like:

```ruby
Vagrant.configure("2") do |config|
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbook.yml"
  end
end
```

Since an Ansible playbook can include many files, you may also collect the related files in
a directory structure like this:

```
$ tree
.
|-- Vagrantfile
|-- provisioning
|   |-- group_vars
|           |-- all
|   |-- playbook.yml
```

In such an arrangement, the `ansible.playbook` path should be adjusted accordingly:

```ruby
Vagrant.configure("2") do |config|
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "provisioning/playbook.yml"
  end
end
```

Vagrant will try to run the `playbook.yml` playbook against all machines defined in your Vagrantfile.

**Backward Compatibility Note**:

Up to Vagrant 1.4, the Ansible provisioner could potentially connect (multiple times) to all hosts from the inventory file.
This behaviour is still possible by setting `ansible.limit = 'all'` (see more details below).

## Additional Options

The Ansible provisioner also includes a number of additional options that can be set,
all of which get passed to the `ansible-playbook` command that ships with Ansible.

* `ansible.extra_vars` can be used to pass additional variables (with highest priority) to the playbook. This parameter can be a path to a JSON or YAML file, or a hash. For example:

    ```
    ansible.extra_vars = {
      ntp_server: "pool.ntp.org",
      nginx: {
        port: 8008,
        workers: 4
      }
    }
    ```
    These variables take the highest precedence over any other variables.
* `ansible.sudo` can be set to `true` to cause Ansible to perform commands using sudo.
* `ansible.sudo_user` can be set to a string containing a username on the guest who should be used
by the sudo command.
* `ansible.ask_sudo_pass` can be set to `true` to require Ansible to prompt for a sudo password.
* `ansible.limit` can be set to a string or an array of machines or groups from the inventory file to further control which hosts are affected. Note that:
  * As Vagrant 1.5, the machine name (taken from Vagrantfile) is set as **default limit** to ensure that `vagrant provision` steps only affect the expected machine. Setting `ansible.limit` will override this default.
  * Setting `ansible.limit = 'all'` can be used to make Ansible connects to all machines from the inventory file.
* `ansible.verbose` can be set to increase Ansible's verbosity to obtain detailed logging:
  * `'v'`, verbose mode
  * `'vv'`
  * `'vvv'`, more
  * `'vvvv'`, connection debugging
* `ansible.tags` can be set to a string or an array of tags. Only plays, roles and tasks tagged with these values will be executed.
* `ansible.skip_tags` can be set to a string or an array of tags. Only plays, roles and tasks that *do not match* these values will be executed.
* `ansible.start_at_task` can be set to a string corresponding to the task name where the playbook provision will start.
* `ansible.raw_arguments` can be set to an array of strings corresponding to a list of `ansible-playbook` arguments (e.g. `['--check', '-M /my/modules']`). It is an *unsafe wildcard* that can be used to apply Ansible options that are not (yet) supported by this Vagrant provisioner. Following precedence rules apply:
  * Any supported options (described above) will override conflicting `raw_arguments` value (e.g. `--tags` or `--start-at-task`)
  * Vagrant default user authentication can be overridden via `raw_arguments` (with custom values for `--user` and `--private-key`)
* `ansible.raw_ssh_args` can be set to an array of strings corresponding to a list of OpenSSH client parameters (e.g. `['-o ControlMaster=no']`). It is an *unsafe wildcard* that can be used to pass additional SSH settings to Ansible via `ANSIBLE_SSH_ARGS` environment variable.
* `ansible.host_key_checking` can be set to `true` which will enable host key checking. As Vagrant 1.5, the default value is `false`, to avoid connection problems when creating new virtual machines.

## Tips and Tricks

### Ansible Parallel Execution

Vagrant is designed to provision [multi-machine environments](/v2/multi-machine) in sequence, but the following configuration pattern can be used to take advantage of Ansible parallelism:

```
  config.vm.define 'machine2' do |machine|
    machine.vm.hostname = 'machine2'
    machine.vm.network "private_network", ip: "192.168.77.22"
  end

  config.vm.define 'machine1' do |machine|
    machine.vm.hostname = 'machine1'
    machine.vm.network "private_network", ip: "192.168.77.21"

    machine.vm.provision :ansible do |ansible|
      ansible.playbook = "playbook.yml"

      # Disable default limit (required with Vagrant 1.5+)
      ansible.limit = 'all'
    end
  end
```

### Provide a local `ansible.cfg` file

Certain settings in Ansible are (only) adjustable via a [configuration file](http://docs.ansible.com/intro_configuration.html), and you might want to ship such a file in your Vagrant project.

As `ansible-playbook` command looks for local `ansible.cfg` configuration file in its *current directory* (but not in the directory that contains the main playbook), you have to store this file adjacent to your Vagrantfile.

Note that it is also possible to reference an Ansible configuration file via `ANSIBLE_CONFIG` environment variable, if you want to be flexible about the location of this file.

### Why Ansible provisioner does connect with a wrong user ?

It is good to know that following Ansible settings always override `config.ssh.username` option defined in [Vagrant SSH Settings](/v2/vagrantfile/ssh_settings.html):

* `ansible_ssh_user` variable
* `remote_user` (or `user`) play attribute
* `remote_user` task attribute

Be aware that copying snippets from Ansible documentation might lead to this problem, as `root` is used as remote user in many [examples](http://docs.ansible.com/playbooks_intro.html#hosts-and-users).

Example of SSH error (with `vvv` log level), where an undefined remote user `xyz` has replaced `vagrant`:

```
TASK: [my_role | do something] *****************
<127.0.0.1> ESTABLISH CONNECTION FOR USER: xyz
<127.0.0.1> EXEC ['ssh', '-tt', '-vvv', '-o', 'ControlMaster=auto',...
fatal: [ansible-devbox] => SSH encountered an unknown error. We recommend you re-run the command using -vvvv, which will enable SSH debugging output to help diagnose the issue.
```
