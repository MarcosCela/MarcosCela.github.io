---
title: "~/.ssh/config and you"
date: 2020-08-31
description: "Simple SSH configuration to make your life easier"
featured_image: "/images/dev/ssh.svg"
categories:
- Tutorial
- SSH
- Configuration
- Linux
tags: ["ssh", "config", "dev"]
include_toc: true
---
# What is exactly ~/.ssh/config?

It's the place for the user-specific configuration for ssh. Like almost any-other CLI tool, [ssh] accepts
a set of configuration parameters by flags, a global config file, or a user config file.


# What is it useful for?

You can configure a lot of parameters for your connections. If you usually connect to one or more hosts
via SSH, it makes things like selecting the identity key, or using a different host alias a breeze.

Under corporate networks it's not unusual to share the same domain for all hosts. You can configure
a common set of parameters for all hosts that match a given domain. As an example, the following
lines in your ***~/.ssh/config*** file:

```text
Host *.my.domain.com
    ServerAliveInterval 30
    IdentityFile ~/.ssh/id_ed25519
    User myuser
    StrictHostKeyChecking yes
```

All hosts that match the host entry "*.my.domain.com" will have the same configuration applied. Additionally,
if you use fully qualified entries, you will be able to (in most shells!) autocomplete the host that you are
trying to connect to. If your config file looks like this:

```text
Host bigserver.my.domain.com
  User admin

Host smallserver.my.domain.com
  User notadmin

Host smallserver.my.domain.com
  User another
```

When you type the following on your shell (... represents when you press ***TAB***):

```shell script
ssh sm...
```

You will be prompted with all host entries that match the given prefix ("sm"). This is specially powerful when
you have several hosts for the same reason.
Additionally, you can set host "aliases" that will just work for your SSH, so you don't have to type
the fully qualified domain name:


```text
Host node1
  HostName node1.my.subdomain.com
```

Typing "ssh node1" will connect us to "node1.my.subdomain.com", saving a lot of typing and guessing
what subdomain you have to use.


Additionally, you can set all sort of automatic redirections, proxying and almost any option that you wish
for. For additional details you can always check the documentation for [ssh_config].






[ssh_config]: https://linux.die.net/man/5/ssh_config
[ssh]: https://linux.die.net/man/1/ssh