# ansible-slv

Ansible is a tool for configuring the content of servers (linux or windows). Terraform is used to automate *infrastructure*, Ansible is used to manage *content* of virtual machines, and Docker is used to manage *content* of containers.

To set up your workstation with Ansible, see the end of this README.

## Run a playbook

The day-to-day operation of Ansible is to run a playbook against an inventory file (`prod`, `dev`)

```bash
## linux
# assuming your localhost user is also your ansible user, and you have your ssh
# key loaded

# Run the jumphosts playbook against the prod servers. Always run with --check first (dry run)
ansible-playbook jumphosts.yml --check
ansible-playbook jumphosts.yml

# Run the jumphosts playbook against a specific server group or specific hostname
ansible-playbook -l jumphost-slv jumphosts.yml --check
ansible-playbook -l jumphost-slv jumphosts.yml
```

```bash
## windows
# we need to provide our Active Directory username and password when we call the playbook
# -k provides a password prompt called `SSH password`, but it is used for kerberos here
ansible-playbook -u pmorahan -k win_example.yml --check
ansible-playbook -u pmorahan -k win_example.yml

## windows bootstrap
# Here we provide the local Administrator password we used to set up the box
ansible-playbook -u Administrator -k bootstrap.yml
```

### Always run \-\-check first

`--check` is Ansible's dry run, and it's good to run it first to avoid surprises.

The roles need to be written to use 'check mode', see `tasks/main.yml` for how this is done. In check mode, green is 'no change', orange is 'will change', and red is 'error'.

Note that not all errors are really *problems* in a dry run - you need to read the log. Some errors are because that item cannot be confirmed because the system is not set up yet. For example, the system can't check the presence of an ssh key on a user if that user does not exist yet. If the user is set up in a previous step, we can then ignore the error on the 'add ssh key' step. TL;DR: read the red text and use your noggin

### External modules/roles

Try and avoid using external modules/roles made by other users. These tend not to be 'generic' and make assumptions about the environment that are often wrong for us. From experience, it takes longer to debug these issues than to just write our own roles, porting in the functionality we want.

Roles written by well-known authors tend to be more suitable for generic use. Just don't use external roles blindly.

### Users on servers

Windows users are expected to be managed via our Active Directory domain - when the server is added to AD, the users should come with it. We don't generally install Windows users via Ansible

Linux users are not provided by Active Directory. The linux users are listed in `group_vars/linux.html` - a linux host needs to be in the `linux` group to get these users.

* `admin_users` have sudo rights and should be installed everywhere
    * sudo rights = being in the `sudo` group
* `dev_users` do not have sudo, and are only installed on hosts that have a var set (eg: jumphost)
* `group_users` and `host_users` do not exist yet, but allow a similar dict to be set in a group/host varfile
   * this allows us to add a specific user list for a specific task/app/server, should we need to

### Ansible Vault

... allows us to save secrets. you'll need a password in `~/.ansible/vault_password` if you don't want to apply it every run

`group_vars` and `host_vars` files encrypted with ansible vault are not readable with regular text tools - you'll need to do `ansible-vault edit PATH/TO/FILE`

## Inventory files, groups, group vars, host vars

At the moment we're just using a single `inventory` file, but we can split this into separate `prod`/`dev`/whatever files at a later date

Each inventory file is made up of a number of groups, and you'll put a host in several groups at the same time. These inventory files don't deal with nested groups very well, and if you duplicate a group name you can get unexpected behaviour as then ansible will pull all group vars in at the same time. Having all groups at the same 'level' in the file helps avoid this problem.

The `group_vars/[group name].yml` file matches the groups listed in the inventory file. Any vars in these files are applied against all members of the matching group

The `host_vars/[host name].yml` file matches the specific hostname, and behaves like the group vars file.


## Bootstrap an unconfigured server

Bootstrapping puts in the *bare minimum* to get a system working with other playbooks. Essentially it installs superadmins and any related initial setup tooling. Regular non-bootstrap playbooks should take care of further functionality - bootstrap commands will often also be in the 'common' role, and anything that needs to be present in general should also be in 'common'. Bootstrap geerally gets run only once, whereas 'common' is repeated.

You will need to put the hostname you're bootstrapping into at least the `bootstrap` and `linux`/`windows` group in the `inventory` file

### Linux

```bash
# the server I'm bootstrapping here has 'admin' as the initial image's
# superuser. I already have the relevant ssh key loaded into my ssh key agent,
# otherwise I could specify the file with --keyfile ~/.ssh/blah.key

# always run with 'check' first, to avoid nasty surprises
ansible-playbook -u admin bootstrap.yml --check

ansible-playbook -u admin bootstrap.yml
```

### Windows

On the windows target, we need to enable the `winrm` protocol. We set this up initially to use basic auth, then we run a bootstrap playbook to add the machine to the domain and switch winrm from basic auth to certificate auth. After that we use normal server playbooks.

```powershell
winrm quickconfig

```

We add the host to be bootstrapped into the inventory file in the 'windows' group, the 'bootstrap' group, and any others that are appropriate. While it's in the bootstrap group, it uses an old, slow auth protocol, which allows us to connect with the local Administrator

#### NTLM vs Kerberos

NTLM is very old, but allows us to connect with the local Administrator account. The bootstrap playbook defaults to this protocol. NTLM is also a very slow protocol, which really slows down Ansible playbook runs as they do a lot of auth'ing.

Kerberos uses our Active Directory accounts and runs at normal speed. Kerberos won't be used while the target server is in the `bootsrap` group in the `inventory` file

#### Hostname 15 character limit

Windows hostnames should be 15 characters or less. When we set the hostname via Ansible and it's longer than 15 characters, an interactive prompt happens that Ansible can't happen. This is Windows asking to truncate the NETBIOS version of the hostname. Haven't figured out how to avoid this yet, so don't exceed 15 chars for now.

----

## Ansible admin / workstation setup

Short form:

1. get yourself a linux install
    * WSL on windows should be fine
2. get your ssh set up
3. get your kerberos/AD set up
4. sort out the vault_pass file
5. start running playbooks

### Windows

Ansible does not support being run from windows. Use Windows Subsystem for Linux (WSL) and install it there following the guide for Linux below. WSL setup is out of scope for this document, but doco is here: https://learn.microsoft.com/en-us/windows/wsl/install

WSL installs Ubuntu by default. The Linux setup below was determined on a Debian box, which is pretty similar.

### Linux

#### Setting up Ansible

* `apt install ansible git`, and clone this repo

For accessing target machines, there is no central 'ansible' or 'deploy' user. For boostrapping (see above) you use the OS's own local admin user. For other playbooks, you use your own personal linux or windows/AD user account, depending on the target machine.

##### Ansible Vault password

Talk to someone who already has it. You can store it in `~/.ansible/vault_pass` and it should be automatically used to decrypt Vault-protected files.

#### Setting up auth for targetting Linux machines

Ansible uses stock ssh, so you will need an ssh key. Another admin will need to add you to the linux users list, which is kept in `group_vars/linux.yml`. The playbook will then need to be run to apply your user to the target servers.

Your **public** ssh key needs to be committed in this repo in `roles/users/files/` - this gets copied to the servers as `/home/USER/.ssh/authorized_keys`, and can hold multiple public keys.

Bootstrapping a linux server happens before we've added our users - for bootstrap, you use the admin user+pass/key that your source image set up. After bootstrap, you use your normal user + ssh key

If you can ssh to a target machine and use `sudo`, you can run an Ansible playbook

#### Setting up auth for targetting Windows machines

For bootstrapping Windows machines, you just need the local Administrator password, and we use the `ntlm` protocol (very slow)

For our other playbooks we use our Active Directory users. Bootstrapping puts the machine onto our AD system, and from there we can use our regular AS users. The trick is getting auth on those users from a linux box that isn't on the domain.

##### Setting up kerberos auth

Before we try to use Ansible, we need to get the ability to auth against kerberos (Active Directory)

* `apt install python-dev libkrb5-dev krb5-user` (also install ansible and git as above)
    * your `pykerberos` library must be v1.2 or later (check via apt or pip3, depending on installation method)
    * `pip install pykerberos --upgrade` fixed my old version installed with apt
* replace the file at `/etc/krb5.conf` with the one in this repo at `conf/kerberos/krb5.conf`
    * if you want to modify your own existing file, these are the mandatory elements:
        * `default_realm` must be in CAPITALS
        * the `[realms]` domain must be in CAPITALS
            * the settings in here must match our domain's nodes, of course
        * there must be an entry for .staff.local in `[domain_realms]`
        * `dns_lookup_realm` likely needs to be false (not sure)
    * there is no service to restart - this file is read each time you try to auth

Once the above is done, you can test your login with the following commands. Get this working before you try to run Ansible against Windows

```bash
# your Active Directory user, with no domain. Enter your usual password when asked
kinit pmorahan

# confirm you have an auth'd ticket that is current/not expired
klist
```

##### Setting up Ansible with Kerberos

After you have `kinit` working, try a test Ansible run with `--check` enabled. This will flush out any additional auth issues (eg: old pykerberos library). Once that is successful, you can now use your Active Directory user against bootstrapped Windows targets

Some Ansible kerberos auth errors and what they meant for us:

```
'Server not found in Kerberos database'
# you're using an IP address in the inventory file
# change `ansible_hostname` to use an FQDN

'kerberos: the specified credentials were rejected by the server'
# your creds are (probably) correct, but not accepted as admin by the target server
# this happens if your user is not in the administrator group - this is an AD issue,
# not ansible/server
# either get yourself into the admins group, or use a different user

'Password incorrect while getting initial credentials'
# you entered a bad password


'kerberos: Bad HTTP response returned from server. Code 500'
# your pykerberos lib is too old, update it
```

---

## Things to try; Tasks TODO

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


### Disable ssh password auth

After bootstrap, ssh keys for admins are on the servers and we should disable ssh password auth. This should be in the common role for linux, but is not currently set - the debian AWS AMIs used so far already have this setting (most AMIs will) but on-prem linux boxes do not have this setting.

This functionality needs to be added, but I haven't gotten around to it yet

### Ensure 'basic auth' is disabled in winrm

It should be by default, but we should have a task in common-windows to ensure it's turned off. NTLM and Kerberos are both encrypted login protocols and should be left enabled.
