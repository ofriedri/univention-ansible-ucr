# Univention Ansible snippets

This repository contains two nice additions to Ansible for managing
Univention Corporate Server systems: providing Univention Config
Registry (UCR) variables as facts and a module for changing Univention
Config Registry variables.


## Univention Config Registry variables as facts

This is a role called `facts` that can be included. It installs a
custom fact file called `ucr.fact` on the managed system and re-loads
facts afterwards.

The `ucr.fact` module itself uses Univention's provided Python module
for accessing the Univention Config Registry. The fact module loads
all UCR variables and dumps all of them as a JSON hash. Ansible itself
reads that output and converts the hash to custom facts. The facts can
be accessed as `ansible_local.ucr['name/of/ucr/variable']`,
e.g. `ansible_local.ucr['server/role']`.

Inside a playbook you should use two plays. The first play should
solely consist of the `facts` role provided in this repository. The
second play should contain everything else. That way all the UCR
variables will be available to all the tasks and roles in the second
play.

For performance reasons the second play should not gather facts.

See the file `example_playbook.yml` for an example how to do this.


## Custom facts for easy distribution handling

When you run Ansible to manage a UCS system, you'll quickly notice
that certain facts about the such as `ansible_facts.distribution`,
`ansible_facts.distribution_release` and
`ansible_facts.distribution_version` aren't correct. Using Ansible's
`debug` module you can quickly see that they're set as follows (all
irrelevant entries skipped):

```
    "distribution": "Debian"
    "distribution_release": "stretch"
    "distribution_version": "4.4-3 errata427"
    "os_family": "Debian"
```

This isn't entirely useful:

1. The `distribution` is actually `Univention`, even though the
   `os_family` is correct.
2. The `distribution_release` should be something like `4.4` which is
   what's used in APT repository sources as code names for the
   releases.
3. The `distribution_version` should also be something like `4.4` for
   easier comparison. If you need the actual patch level or errata
   numbers, you can obtain them from the `ansible_local.ucr` facts
   mentioned above (`ansible_local.ucr['version/patchlevel']` and
   `ansible_local.ucr['version/erratalevel']`).

For this reason the `facts` role mentioned above sets three custom
facts that'll allow easier distinction between distributions in
subsequent roles & tasks. These variables are set on UCS systems as
well as on non-UCS systems. On UCS systems they're set to UCS-relevant
values whereas they're set to Ansible's corresponding values on
non-UCS systems. Examples:

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

See the file `example_playbook.yml` for how these custom facts can be
used for differentiating between distributions in an easy-to-read
manner.

Note that the existing facts `ansible_facts.distribution…` are _not_
modified. This is due to the `set_fact` module not being able to
overwrite them. That module _can_ overwrite the global variants
`ansible_facts_distribution…`, but not the one inside the
`ansible_facts` hash. On top of that the global facts only exist if
the `ansible.cfg` setting `inject_facts_as_vars` is set to `True`.


## Modifying Univention Config Variables

This is an Ansible module called `univention_config_registry`. It can
be used to set or unset one or more UCR variables in a given task. If
actions are associated with the keys (e.g. rebuilding `/etc/aliases`
when one of the variables `mail/aliases/…` is changed), those actions
will be executed directly after the modification, similar to how
they'd be executed if the admin ran the `ucr set…` command line
utility.

The module itself contains documentation on how to use the module.

The module supports Ansible's check mode.

In order for Ansible to find the module, your `ansible.cfg` must
include the module's path in the `library` setting in the `[defaults]`
section. Example for the layout used in this repository:

```
[defaults]
library = ./library
```

See the file `example_playbook.yml` which uses the module to set the
Postfix mail alias for `root`.

## Copyright

Copyright (c) 2020 Moritz Bunkus <m.bunkus@linet-services.de>

## License: CC0

To the extent possible under law, Moritz Bunkus has waived all
copyright and related or neighboring rights to Univention Ansible
snippets. This work is published from: Germany.
