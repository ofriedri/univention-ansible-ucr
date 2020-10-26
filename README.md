Role Name
=========

ucr-facts

Description
===========

Create ansible local facts for Univention Config Registry variables. It includes a module that allows reading and setting these variables. The module is available after the ucr-facts role has been called

Requirements
============

This role and module are designed to run against the Univention Corporate Server, aka UCS.

Module Name
===========

univention\_config\_registry

Module Parameters
=================

* `keys` - A dict of keys to set or unset. In case of unsetting, the values are ignored. Either this or 'kvlist' must be given.
* `kvlist`- You pass in a list of dicts with this parameter instead of using a dict via 'keys'. Each of the dicts passed via 'kvlist' must contain the keys 'key' and 'value'. This allows the use of Jinja in the UCR keys to set/unset.
* `state` - Either 'present' for setting the key/value pairs given with 'keys' or 'absent' for unsetting the keys from the 'keys' dict. Default is 'present'.


Dependencies
============

Univention config\_registry python module, which is already installed on UCS systems

Example Playbooks
=================

The role can be used like this:
```
    - hosts: servers
      roles:
         - ucr-facts
```

Afterwards the module `univention_config_registry` can be used like this:
```
# Set various keys
- name: Set proxy configuration
  univention_config_registry:
    keys:
      proxy/http: http://myproxy.mydomain:3128
      proxy/https: http://myproxy.mydomain:3128

# Alternative syntax with use of Jinja.
- name: Set /etc/hosts entries
  univention_config_registry:
    kvlist:
      - key: "hosts/static/{{ item }}"
        value: myhost.fqdn
    loop: [ '192.168.0.1', '192.168.1.1' ]

# Clear proxy configuration
- name: Do not use a proxy
  univention_config_registry:
    keys:
      proxy/http:
      proxy/https:
    present: absent
```

## In-Depth Description

### Univention Config Registry variables as facts

This role installs a custom fact file on the managed systenm, called `ucr.fact`
and subsequently refreshes the facts known to ansible to make them availabe right
away.

The `ucr.fact` file is a script, written in python 2,that uses Univention's
provided Python module to access the Univention Config Registry. The fact
module loads all UCR variables and outputs them as a JSON dictionary. Ansible
reads the output and converts it to custom facts. The facts can be addressed
using `ansible_local.ucr['name/of/ucr/variable']` syntax, for instance
`ansible_local.ucr['server/role']` would access the `server/role` UCR variable.

Inside a playbook it's advisable to use two plays. The first play should
consist of the `facts` role provided in this repository only. The
second play then contains the rest of your tasks. That way all the UCR
variables will be available to any task or role that is called from the second
play

The second play can ommit gathering facts as that has been done already. This
will also be advantageous to this play's execution time.

An example playbook is available at the end of this file.

### Custom facts for easy distribution handling

Per default ansible sees an Univention Corporate Server instance as a Debian
system as that's the Linux distribution the UCS has been forked off. The
relevant ansible facts will look like this:
```
    "distribution": "Debian"
    "distribution_release": "stretch"
    "distribution_version": "4.4-3 errata427"
    "os_family": "Debian"
```

This isn't entirely useful:

1. The `distribution` actually is `Univention`, even though the
   `os_family` is correct.
2. The `distribution_release` should look something like `4.4` which is
   what's being used in APT repository sources as code names for the UCS
   releases.
3. The same is true for `distribution_version`, which should be the UCS
   version (`4.4`for example) instead of the Debian version for easier
   comparability. patch level and errata numbers now can be obtained
   from the `ansible_local.ucr` facts mentioned above
   Those can be found under `ansible_local.ucr['version/patchlevel']` and
   `ansible_local.ucr['version/erratalevel']`.

In order to rectify this this role sets three custom facts that allow for
easier distinction between UCS distributions in subsequent roles & tasks.
On UCS systems they're set to UCS-relevant values. In case you run this role
against a non-UCS system these are set to ansible's distribution variables.

#### Examples

`my_facts_…` on a UCS system:

```
    "my_facts_distribution": "Univention"
    "my_facts_distribution_release": "4.4"
    "my_facts_distribution_version": "4.4"
```

`my_facts_…` on an Ubuntu system:

```
    "my_facts_distribution": "Ubuntu"
    "my_facts_distribution_release": "bionic"
    "my_facts_distribution_version": "18.04"
```

The example playbook at the end of this file shows how these custom facts can
be used for differentiating between distributions in an easy-to-read manner.

Note that the pre-existing facts `ansible_facts.distribution…` are _not_
modified in any way. The `set_fact` module is not able to overwrite them.
2The embedded module _could overwrite the global counter-part
`ansible_facts_distribution…`, but not the one inside the
`ansible_facts` hash. In aqddition to that the global facts only exist if
the `ansible.cfg` setting `inject_facts_as_vars` is set to `True`.

### Modifying Univention Config Variables

The embedded module named `univention_config_registry` can be used after the
role has been called. It is possible to copy the file residing in library/ into
ansible's module search path (or to modify the `default.library` setting in
ansible.fg accordingly), which makes it usable independently from the role.

It can be used to set or unset one or more UCR variables
in a given task. If actions are triggered by changing the keys (e.g. rebuilding
`/etc/aliases` when one of the variables `mail/aliases/…` is changed), those
actions will be executed immediately after the modification has taken place,
similar to how they'd be executed if the admin had run the `ucr set…` command
line utility.

The module itself contains documentation on how to use it, which can be
displayed using ansible-doc.

The module supports ansible's check mode.

For an example for its usage See the example playbook at the end of this file
which uses the module to set the Postfix mail alias for `root`.

## Copyright

Copyright (c) 2020 
* Moritz Bunkus <m.bunkus@linet-services.de>
* Oliver Friedrich <friedrich@univention.de>
* Sascha Schneider <sascha.schneider@univention.de>

## License

GNU General Public License v3.0

## Example Playbook

```
# This example playbook contains of two sections.
#
# The first section installs the custom "ucr.fact" file even in check
# mode (!) on the managed machine. Afterwards it re-reads the
# facts. That way the contents of the Univention Config Registry will
# be available as facts if the managed system is a Univention system.
#
# Additionally a couple of facts called "my_facts_…" will be set that
# will make checking for Univention easier.
#
# The second section adjusts the entry for "root" in the /etc/aliases
# file. On Univention systems it sets the appropriate UCR variable
# using the our own "univention_config_registry" module. On other
# systems it will modify the file directly. The distinction between
# the two cases is made by checking the aforementioned "my_facts_…"
# facts.

---
- name: "gather custom facts"
  hosts: "all"
  gather_facts: true

  roles:
    - ucr-facts

- name: "setup root mail alias"
  hosts: "all"
  gather_facts: false

  # Ensure Postfix sends mail to 'root' to correct address.
  tasks:
    - when: "my_facts_distribution == 'Univention'"
      name: "mail alias for root is set on Univention"
      univention_config_registry:
        keys:
          mail/alias/root: "root@my.dom.ain"

    - when: "my_facts_distribution != 'Univention'"
      block:
        - name: "mail alias for root is set on non-Univention"
          lineinfile:
            name: "/etc/aliases"
            regexp: "^root:.*"
            line: "root: root@my.dom.ain"
          register: "output_"

        - when: "output_.changed"
          name: "rebuild aliases database"
          command: "postalias /etc/aliases"
```
