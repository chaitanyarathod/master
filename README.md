# ansible-slv

Ansible is a tool for configuring the content of servers (linux or windows).

The servers themselves can be launched with Terraform (infrastructure config), but it can't handle content well - which apps are installed, or how their config files looks, or which users are enabled, so on and so forth.

To set up your workstation with Ansible, see the end of this README. To set up a brand new machine, see the 'bootstrap' section.

## Run a playbook

The day-to-day operation of Ansible is to run a playbook against an inventory file (`prod`, `dev`)

```bash
## linux
# assuming your localhost user is also your ansible user, and you have your ssh
# key loaded

# Run the jumphosts playbook against the prod servers. Always run with --check first (dry run)
ansible-playbook -i prod jumphosts.yml --check
ansible-playbook -i prod jumphosts.yml

# Run the jumphosts playbook against a specific server group or specific hostname
ansible-playbook -i prod -l jumphost-slv jumphosts.yml --check
ansible-playbook -i prod -l jumphost-slv jumphosts.yml
```

### Always run \-\-check first

`--check` is Ansible's dry run, and it's good to run it first to avoid surprises.

The roles need to be written to use 'check mode', see `tasks/main.yml` for how this is done. In check mode, green is 'no change', orange is 'will change', and red is 'error'.

Note that not all errors are really *problems* in a dry run - you need to read the log. Some errors are because that item cannot be confirmed because the system is not set up yet. For example, the system can't check the presence of an ssh key on a user if that user does not exist yet. If the user is set up in a previous step, we can then ignore the error on the 'add ssh key' step. TL;DR: read the red text and use your noggin

## Inventory files

We have two inventory files, `prod` and `dev`. They're separated just as an extra layer of safety - some changes can be experimented with in the dev file, and then ported across to prod when ready.

The inventory file has a weird layer called 'children'. This is a hangover from the .ini format of the inventory file, and simply means that the subelements of this group are also groups themselves. Just ignore it.

When adding a new host, ensure it's in all the groups that are relevant to it in the inventory file.

### Flat 'group' structure

The groups themselves can be nested, but we're using a flat approach. This means that adding a single new host should go in several groups - all that are relevant to the host. If we used a nested approach to groups, we would need to be careful that all group names were unique, since Ansible will read *all* groups of the requested name for variables. Example: I had a nested arrangement with 'webservers' in both the 'linux' and 'windows' groups. Ansible was reading both entries for variables, and then trying to connect to the linux server with a windows protocol.

### Dynamic inventory

We don't currently use this, but it is possible to set up Ansible to query your IaaS for VMs to configure, rather than have a predetermined list.

## External modules/roles

Try and avoid using external modules/roles made by other users. These tend not to be 'generic' and make assumptions about the environment that are often wrong for us. From experience, it takes longer to debug these issues than to just write our own roles, porting in the functionality we want.

Roles written by well-known authors tend to be more suitable for generic use. Just don't use external roles blindly.

## Linux vs Windows

When making a new role, split actions into separate linux and windows segments in the `tasks/main.yml` file. See the bootstrap role for how to do this.

This should still be done for consistency when the role can only be used in one or the other (eg: installing a toolchain that only runs on one)

### Users on servers

Windows users are expected to be managed via our Active Directory domain - when the server is added to AD, the users should come with it.

Linux users are not currently planned to be domain-provided. There are dicts in `group_vars/linux.html` relating to users

* `admin_users` have sudo rights and should be installed everywhere
    * sudo rights = being in the `sudo` group
* `dev_users` do not have sudo, and are only installed on hosts that have a var set (eg: jumphost)
* `group_users` and `host_users` do not exist yet, but allow a similar dict to be set in a group/host varfile
   * this allows us to add a specific user list for a specific task/app/server, should we need to

If a particular dev user needs sudo on a particular server, this setup does not currently handle it. You could add the sudo group to their user, but that will give them sudo everywhere. Figure it out when we have a need.

## Inventory files, groups, group vars, host vars

The inventory files `prod` and `dev` are separate to make things a little safer. Do experimental work on `dev` servers, and then port the setup to `prod` servers.

Each inventory file is made up of a number of groups, and you'll put a host in several groups at the same time. These inventory files don't deal with nested groups very well, and if you duplicate a group name you can get unexpected behaviour as then ansible will pull all group vars in at the same time. Having all groups at the same 'level' in the file helps avoid this problem.

The `group_vars/[group name].yml` file matches the groups listed in the inventory file. Any vars in these files are applied against all members of the matching group

The `host_vars/[host name].yml` file matches the specific hostname, and behaves like the group vars file.

## Ansible Vault

allows us to save secrets. you'll need a password in `~/.ansible/vault_password` if you don't want to apply it every run

`group_vars` and `host_vars` files encrypted with ansible vault are not readable with regular text tools - you'll need to do `ansible-vault edit PATH/TO/FILE`

## Bootstrap an unconfigured server

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

## Ansible workstation setup
### Linux

You will need:

1. your own ssh key
    * once users are added to a machine, this is the key you connect with
2. ssh keys for anything that needs to be bootstrapped
    * eg: when you create an EC2 server, you specify an ssh key to use with it
    * this key is used to run the bootstrap playbook, which adds admin users
3. ansible vault password in `~/.ansible/vault_password`, to access encrypted vars files
4. ansible installed (2.10+)

### Windows

dunno yet


## Things to try

### Adding a user

The windows users will usually be part of adding the server to our Active Directory domain.

The linux users are kept in a dict in `group_vars/linux.yml`. Try adding a new user

1. copy an existing user
2. change the name of the section
3. change the `uid` to the next one up
4. switch the password hash for the default one, or a known one for the user
5. get the user's ssh public key, and put it into the `roles/users/files` directory
    * this file can list multiple public keys - it becomes the `.ssh/authorized_keys` file on the target
6. run Ansible against the sandbox machine with `--check`
7. run Ansible against the sandbox machine for real
8. log into the sandbox and see if your user is there
9. edit the new user to toggle the 'remove' field
10. rerun Ansible against the sandbox machine to remove that user
