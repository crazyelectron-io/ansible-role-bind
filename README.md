# ansible-role-bind

Ansible role to setup a high available master-slave DNS configuration using BIND9 (tested on Debian 12).

> Be aware that this role is opinionated and fits my preferences and way of working.
> It may or may not be suitable for your needs.

## Role variables

### Mandatory variables

`rndc_key` must be specified for the secure Dynamic DNS update (used by DHCP).

`search_domain` must be defined to create an entry in the Debian `/etc/resolv.conf` file and configure this domain as authorative zone on the DNS servers.

These variables can be specified in the `all.sops.yaml` inventory file (prefered) or in the `vars`section of an Ansible playbook, like this:

```yaml
- hosts: all
  become: true
  gather_facts: true
  vars:
    bind9_forward: false
    search_domain: example.com
  roles:
    ...
    - bind
    - dhcp
  ...
```

### Important optional parameters (with defaults)

`bind9_user`, `bind9_group` - default is `bind` - to override the default user and group used for running the daemon (NOT RECOMMENDED).

`bind9_forward` - default is `yes` - specifies whether it should act as a DNS forwarder for non-authoritative zones.

`bind9_forward_servers` - default is `8.8.8.8` and `1.1.1.1` and can be overriden as needed.

`search_domain` - default is `example.com` - specifies the main domain used for the Bind configuration and zone files. Currently only one domain is supported in this playbook.

`bind9_log_severity` - default is `info` - possible values are `critical | error | warning | notice | info | debug [ level ] | dynamic`.

`internal_networks` - default is `127.0.0.0/8`, `10.0.0.0/8` and `192.168.0.0/16` and can be overriden as needed. For instance:

```yaml
internal_networks:
  - 10.10.0.0/22
  - 10.20.0.0/24
  - 10.60.10.0/24
```

`install_zone_files` - default is false - indicate whether the domain zone files must be reapplied (losing Dynamic DNS registrations from DHCP)`

`bibnd9_zone_entries` - default is `[]` - list of zone file entries to deploy for the main domain. Example:

```yaml
bind9_zone_entries:
    - host: shelob
      rtype: A
      address: 192.168.0.1
    - host: frodo
      rtype: A
      address: "{{my_address}}"
    - host: gandalf
      rtype: A
      address: 192.168.2.2
    - host: sauron
      rtype: CNAME
      address: gandalf
```

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

1. Add the bind dependancy to `requirements.yaml`.
2. Download the external roles: `ansible-galaxy install -r roles/requirements`
3. Add a `.gitignore` in the `roles` directory to ensure this downloaded roles are not committed to the playbook repository (see `.gitignore-example`).
4. Add the role variables to `inventorygroup_vars\all.sops.yaml` (see `example.all.sops.yaml`).
5. Setup the inventory hosts file `hosts.yaml` (see `example.hosts.yaml`).
6. Launch your playbook: `ansible-playbook -i inventory some_playbook.yml [-u ANSIBLE_USER]`

Zone file template:

```yaml
; {{ansible_managed}}
; synopsis: Zone file for the internal network DNS
$ORIGIN .
$TTL 28800      ; 8 hours
{{ internal_domain }}   IN SOA  dns01.{{search_domain}}. root.{{search_domain}}. (
                                {{bind9_zone_serial}} ; serial
                                3600       ; refresh (1 hour)
                                600        ; retry (10 minutes)
                                86400      ; expire (1 day)
                                30000      ; minimum (8 hours 20 minutes)
                                )
                        NS      dns01.{{search_domain}}.
                        NS      dns02.{{search_domain}}.
$ORIGIN {{search_domain}}.
$TTL 3600
server1                 A       10.0.0.1
server2                 A       10.100.0.2
dns01                   A       10.0.0.3
```

And reverse zone file template:

```yaml
$ORIGIN .
$TTL 3600      ; 1 hour
10.in-addr.arpa         IN SOA  dns01.{{search_domain}}. root.{{search_domain}}. (
                                {{bind9_zone_serial}}  ; serial
                                604800     ; refresh (1 week)
                                86400      ; retry (1 day)
                                2419200    ; expire (4 weeks)
                                604800     ; minimum (1 week)
                                )
                        NS      dns01.
$ORIGIN 10.in-addr.arpa.
$TTL 3600
10.0.0.1                PTR     server1.{{search_domain}}.
10.100.0.2              PTR     server2.{{search_domain}}.
```
