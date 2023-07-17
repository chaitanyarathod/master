# ansible-slv

# Run a playbook

```bash
# assuming your localhost user is also your ansible user, and you have your ssh
# key loaded

# Run the jumphosts playbook against the prod servers
ansible-playbook -i prod jumphosts.yml

# Run the jumphosts playbook against a specific server group or specific hostname
ansible-playbook -i prod -l jumphost-slv jumphosts.yml
```


# Bootstrap an unconfigured server

Bootstrapping puts in the bare minimum to get a system working with other playbooks. Essentially it installs superadmins and any related tooling. Regular non-bootstrap playbooks should take care of further functionality.

You will need to put the hostname you're bootstrapping into at least the `bootstrap` and `linux`/`windows` group

```bash
# the server I'm bootstrapping here has 'admin' as the initial image's
# superuser. I already have the relevant ssh key loaded into my ssh key agent,
# otherwise I could specify the file with --keyfile ~/.ssh/blah.key

# always run with 'check' first, to avoid nasty surprises
ansible-playbook -i prod -u admin bootstrap.yml --check

ansible-playbook -i prod -u admin bootstrap.yml
```

# Linux vs Windows

When making a new role, split actions into separate linux and windows segments in the `tasks/main.yml` file. See the bootstrap role for how to do this.

This should still be done for consistency when the role can only be used in one or the other (eg: installing a toolchain that only runs on one)

# Inventory files, groups, group vars, host vars

The inventory files `prod` and `dev` are separate to make things a little safer. Do experimental work on `dev` servers, and then port the setup to `prod` servers.

Each inventory file is made up of a number of groups, and you'll put a host in several groups at the same time. These inventory files don't deal with nested groups very well, and if you duplicate a group name you can get unexpected behaviour as then ansible will pull all group vars in at the same time. Having all groups at the same 'level' in the file helps avoid this problem.

The `group_vars/[group name].yml` file matches the groups listed in the inventory file. Any vars in these files are applied against all members of the matching group

The `host_vars/[host name].yml` file matches the specific hostname, and behaves like the group vars file.

## Ansible Vault

allows us to save secrets. you'll need a password in `~/.ansible/vault_password` if you don't want to apply it every run

group_vars and host_vars files encrypted with ansible vault are not readable with regular text tools - you'll need to do `ansible vault edit` (TODO: I think that's the command)


# Workstation setup
## Linux

1. your own ssh key
2. ssh keys for anything that needs to be bootstrapped
3. ansible vault password in `~/.ansible/vault_password`
4. ansible installed (2.10+)

## Windows

dunno yet

