---
description: |
    Communicators are the mechanism Packer uses to upload files, execute
    scripts, etc. with the machine being created.
layout: docs
page_title: Communicators
sidebar_current: 'docs-communicators'
---

# Communicators

Communicators are the mechanism Packer uses to upload files, execute scripts,
etc. with the machine being created.

Communicators are configured within the
[builder](/docs/templates/builders.html) section. Packer currently supports
three kinds of communicators:

-   `none` - No communicator will be used. If this is set, most provisioners
    also can't be used.

-   [ssh](/docs/communicators/ssh.html) - An SSH connection will be established to the machine. This is
    usually the default.

-   [winrm](/docs/communicators/winrm.html) - A WinRM connection will be established.

In addition to the above, some builders have custom communicators they can use.
For example, the Docker builder has a "docker" communicator that uses
`docker exec` and `docker cp` to execute scripts and copy files.

For more details on how to use each communicator, click the links above to be
taken to each communicator's page.
