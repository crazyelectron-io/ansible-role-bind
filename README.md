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

This repo has all the secrets in one YAML file `./inventory/group_vars/all.sops.yaml` and the very first time we create it, it must be encrypted manually once with SOPS:

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
