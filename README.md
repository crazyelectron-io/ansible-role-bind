# ansible-role-bind

Ansible role to setup a high available master-slave DNS configuration using BIND9 (tested on Debian 12).

> Be aware that this role is opinionated and fits my preferences and way of working.
> It may or may not be suitable for your needs.

## Role variables

### Mandatory variables

`rndc_key` must be specified for the secure Dynamic DNS update (used by DHCP).

`main_domain` must be defined to create an entry in the Debian `/etc/resolv.conf` file and configure this domain as local zone on the DNS servers.

These variables can be specified in the `all.sops.yaml` inventory file or in the `vars`section of an Ansible playbook, like this:

```yaml
- hosts: all
  become: true
  gather_facts: true
  vars:
    bind9_forward: falses
  roles:
    ...
    - bind
    - dhcp
  ...
```

### Important optional parameters (with defaults)

`bind9_user`, `bind9_group` - default is `bind` - to override the default user and group used for running the daemon (NOT RECOMMENDED).

`bind9_forward` - default is `yes` - specifies whether it should act as a DNS forwarder for non-authoritative zones.

`bind9_forward_servers` - default is `1.1.1.1` and `8.8.8.8` and can be overriden as needed.

`bind9_log_severity` - default is `info` - possible values are `critical | error | warning | notice | info | debug [ level ] | dynamic`.

`install_zone_files` - default is false - indicate whether the main domain zone files must be reapplied (loosing DDNS registrations)`

## Encrypted configuration file

This role uses global variables from `./inventory/group_vars/all.sops.yaml` and the very first time we create it, it must be encrypted manually once with SOPS:

```bash
sops -e -i ./inventory/group_vars/all.sops.yaml
```

After that the VS Code extension `signageos/vscode-sops` automattically decrypts and reencrypts the file whenever it is edited (and Ansible has a plugin to decrypt variables when loaded - specified in `ansible.cfg`).
For this to work we need to have `sops` and `age` installed and a key generated and stored in `~/.sops/key,txt`.
In addition, a SOPS configuration file is created in the `./inventory` directory which contains the `age` public key:

```yaml
  # file: ./inventory/.sops.yaml
  creation_rules:
    - path_regex: .*vars/.*
      age: "age109fzapgarv59gpxu5zqmwgn8j7hxmfz8dhrz9lrqvky046jxafmse38kvj"
```

## Inventory example

```yaml
# file: inventory/group_vars/all.sops.yaml
dns_master:
  hosts:
    gandalf:
      ansible_port: 22
      ansible_host: 10.0.0.10
dns_slave:
  hosts:
    sauron:
      ansible_port: 22
      ansible_host: 10.0.0.8
dns_server:
  children:
    dns_master:
    dns_slave:
```

### Roles directory

To ensure the local roles that are specific for the playbook are dfistinguishable, place them in the directory `roles/local` to ensure onlyt hese are commited to the repository.
Roles like this that are downloaded via `requirements.yaml` are placed directly in the `roles` directory and not included in the playbook repository.
A simple `roles/.gitignore` will take care of that:

```ini
#Ignore everything in roles dir...
/*
# ... but current file...
!.gitignore
# ... external role requirement file
!requirements.yml
# ... and configured custom/local roles
!local*/
```

### Ansible configuration

To use the automated encryption/decryption, there are a few additional requirements for the Ansible configuration to be specified in the `ansible.cfg` file.

```ini
[defaults]
...
vars_plugins_enabled = host_group_vars,community.sops.sops
roles_path = roles
[community.sops]
vars_stage = inventory
[inventory]
enable_plugins = host_list, script, auto, yaml, ini
```

## Usage of this role

To use this role, include the following section in a `requirements.yml` file in the local `roles` directory:

```yaml
# Include the 'bind` role from GitHub
- src: git@github.com:crazyelectron-io/ansible-role-bind.git
  scm: git
  version: main
  name: bind
```

> Only include the 'top' roles, dependencies - when listed in `meta/main.yml` of the imported role - will be downloaded automatically.

To retrieve roles like this in your project, run `ansible-galaxy install -r requirements.yaml`.
Because these roles will not be updated locally when the repository is changed, to refresh an already retrieved role use `ansible-galaxy install -f -r requirements.yaml`

## Dependencies

None.

## Workflow to deploy from the project

1. Add the bind dependancy to `rerquirements.yaml`.
2. Download the external roles: `ansible-galaxy install -r roles/requirements`
3. Add a `.gitignore` in the `roles` directory to ensure this downloaded roles are not committed to the playbook repository (see `.gitignore-example`).
4. Add the role variables to `inventorygroup_vars\all.sops.yaml` (see `example.all.sops.yaml`).
5. Setup the inventory hosts file `hots.yaml` (see `example.hosts.yaml`).
6. Launch your playbook: `ansible-playbook -i inventory some_playbook.yml [-u ANSIBLE_USER]`

## Zone configuration in separate role

Example Ansible tasks:

```yaml
---
- set_fact:
    update_zone: false

- block:
    - name: DNS | Get stats of the target zone file
      ansible.builtin.stat:
        path: "{{ bind9_lib_path }}/db.{{ internal_domain }}"
      register: zone_stats
    - name: DNS | Get stats of the template zone file
      ansible.builtin.stat:
        path: "{{ role_path }}/templates/zones/internal-domain.zone.j2"
      register: template_stats
      become: false
      delegate_to: localhost
    - name: DNS | Check if the zone file is newer than the template
      set_fact:
        update_zone: true
      when: not zone_stats.stat.exists or zone_stats.stat.mtime < template_stats.stat.mtime
  when: inventory_hostname in groups.dns_master

- name: Create or update the internal domain {{ internal_domain }} zone files
  block:
    - include_tasks: serialnumber.yaml

    - name: DNS | Freeze the internal domain {{ internal_domain }} zones
      ansible.builtin.shell: 'rndc freeze {{ internal_domain }}'
      ignore_errors: true

    - name: DNS | Copy internal domain {{ internal_domain }} zone file
      template:
        src: zones/internal-domain.zone.j2
        dest: "{{ bind9_lib_path }}/db.{{ internal_domain }}"
        owner: "{{ bind9_user }}"
        group: "{{ bind9_group }}"
        mode: 0644
      register: resync_zone

    - name: DNS | Copy internal domain {{ internal_domain }} reverse zone file
      template:
        src: zones/internal-domain.rev-zone.j2
        dest: "{{ bind9_lib_path }}/db.{{ internal_domain }}.rev"
        owner: "{{ bind9_user }}"
        group: "{{ bind9_group }}"
        mode: 0644
      register: resync_rev_zone

  always:
    - name: DNS | Resync {{ internal_domain }} zone journal when files are updated
      ansible.builtin.shell: 'rndc sync -clean {{ internal_domain }}'
      when: resync_zone.changed | bool or resync_rev_zone.changed | bool

    - name: DNS | Reload the internal domain {{ internal_domain }} zone files
      ansible.builtin.shell: 'rndc reload {{ internal_domain }}'
      ignore_errors: true

    - name: DNS | Unfreeze the internal domain {{ internal_domain }} zone files
      ansible.builtin.shell: 'rndc thaw {{ internal_domain }}'
      ignore_errors: true

  when: inventory_hostname in groups.dns_master and update_zone | bool

- block:
  - name: "DNS | Get services info"
    ansible.builtin.service_facts:
  - name: DNS | Fail if named is not running
    ansible.builtin.fail:
      msg: "DNS configuration failed, service not running!"
    when: services['named'].state != 'running'
  when: inventory_hostname in groups.dns_servers

```

Serialnumber generator:

```yaml
---
- name: "{{ role_path|basename }} | get unix time"
  shell: echo $(date +%s)
  register: unix_time_stamp
  delegate_to: localhost
  run_once: true
  become: false
  changed_when: false

- name: "{{ role_path|basename }} setting execution facts"
  set_fact:
    bind9_zone_serial: "{{ unix_time_stamp.stdout }}"
  run_once: true
  become: false
```

Zone file template:

```yaml
; {{ ansible_managed }}
; synopsis: Zone file for the internal network domain
$ORIGIN .
$TTL 28800      ; 8 hours
{{ internal_domain }}   IN SOA  dns01.{{ internal_domain }}. root.{{ internal_domain }}. (
                                {{ bind9_zone_serial }} ; serial
                                3600       ; refresh (1 hour)
                                600        ; retry (10 minutes)
                                86400      ; expire (1 day)
                                30000      ; minimum (8 hours 20 minutes)
                                )
                        NS      dns01.{{ internal_domain }}.
                        NS      dns02.{{ internal_domain }}.
$ORIGIN {{ internal_domain }}.
$TTL 3600
server1                 A       10.0.0.1
server2                 A       10.100.0.2
dns01                   A       10.0.0.3
```

And reverse zone file template:

```yaml
$ORIGIN .
$TTL 3600      ; 1 hour
10.in-addr.arpa         IN SOA  dns01.{{ main_domain }}. root.{{ main_domain }}. (
                                {{ bind9_zone_serial }}  ; serial
                                604800     ; refresh (1 week)
                                86400      ; retry (1 day)
                                2419200    ; expire (4 weeks)
                                604800     ; minimum (1 week)
                                )
                        NS      dns01.
$ORIGIN 10.in-addr.arpa.
$TTL 3600
10.0.0.1                PTR     server1.{{ main_domain }}.
10.100.0.2              PTR     server2.{{ main_domain }}.
```
