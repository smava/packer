---
description: |
    The Vagrant Packer builder is able to launch Vagrant boxes and
    re-package them into .box files
layout: docs
page_title: 'Vagrant - Builders'
sidebar_current: 'docs-builders-vagrant'
---

# Vagrant Builder

The Vagrant builder is intended for building new boxes from already-existing
boxes. Your source should be a URL or path to a .box file or a Vagrant Cloud
box name such as `hashicorp/precise64`.

Packer will not install vagrant, nor will it install the underlying
virtualization platforms or extra providers; We expect when you run this
builder that you have already installed what you need.

By default, this builder will initialize a new Vagrant workspace, launch your
box from that workspace, provision it, call `vagrant package` to package it
into a new box, and then destroy the original box. Please note that vagrant
will *not* remove the box file from your system (we don't call
`vagrant box remove`).

You can change the behavior so that the builder doesn't destroy the box by
setting the `teardown_method` option. You can change the behavior so the builder
doesn't package it (not all provisioners support the `vagrant package` command)
by setting the `skip package` option. You can also change the behavior so that
rather than initializing a new Vagrant workspace, you use an already defined
one, by using `global_id` instead of `source_box`.

## Configuration Reference

### Required:

-   `source_path` (string) - URL of the vagrant box to use, or the name of the
    vagrant box. `hashicorp/precise64`, `./mylocalbox.box` and
    `https://example.com/my-box.box` are all valid source boxes. If your
    source is a .box file, whether locally or from a URL like the latter example
    above, you will also need to provide a `box_name`. This option is required,
    unless you set `global_id`. You may only set one or the other, not both.

    or

-   `global_id` (string) - the global id of a Vagrant box already added to Vagrant
    on your system. You can find the global id of your Vagrant boxes using the
    command `vagrant global-status`; your global\_id will be a 7-digit number and
    letter combination that you'll find in the leftmost column of the
    global-status output. If you choose to use `global_id` instead of
    `source_box`, Packer will skip the Vagrant initialize and add steps, and
    simply launch the box directly using the global id.

### Optional:

-   `output_dir` (string) - The directory to create that will contain
    your output box. We always create this directory and run from inside of it to
    prevent Vagrant init collisions. If unset, it will be set to `packer-` plus
    your buildname.

-   `box_name` (string) - if your source\_box is a boxfile that we need to add
    to Vagrant, this is the name to give it. If left blank, will default to
    "packer\_" plus your buildname.

-   `provider` (string) - The vagrant [provider](docs/post-processors/vagrant.html).
    This parameter is required when `source_path` have more than one provider,
    or when using `vagrant-cloud` post-processor. Defaults to unset.

-   `checksum` (string) - The checksum for the .box file. The type of the
    checksum is specified with `checksum_type`, documented below.

-   `checksum_type` (string) - The type of the checksum specified in `checksum`.
    Valid values are `none`, `md5`, `sha1`, `sha256`, or `sha512`. Although the
    checksum will not be verified when `checksum_type` is set to "none", this is
    not recommended since OVA files can be very large and corruption does happen
    from time to time.

-   `template` (string) - a path to a golang template for a
    vagrantfile. Our default template can be found
    [here](https://github.com/hashicorp/packer/blob/master/builder/vagrant/step_create_vagrantfile.go#L23-L37). So far the only template variables available to you are {{ .BoxName }} and
    {{ .SyncedFolder }}, which correspond to the Packer options `box_name` and
    `synced_folder`.

    You must provide a template if your default vagrant provider is Hyper-V.
    Below is a Hyper-V compatible template.

    ``` ruby
    Vagrant.configure("2") do |config|
        config.vm.box = "{{ .BoxName }}"
        config.vm.network 'public_network', bridge: 'Default Switch'
    end
    ```

-   `skip_add` (bool) - Don't call "vagrant add" to add the box to your local
    environment; this is necessary if you want to launch a box that is already
    added to your vagrant environment.

-   `teardown_method` (string) - Whether to halt, suspend, or destroy the box when
    the build has completed. Defaults to "halt"

-   `box_version` (string) - What box version to use when initializing Vagrant.

-   `add_cacert` (string) - Equivalent to setting the
    [`--cacert`](https://www.vagrantup.com/docs/cli/box.html#cacert-certfile)
    option in `vagrant add`; defaults to unset.

-   `add_capath` (string) - Equivalent to setting the
    [`--capath`](https://www.vagrantup.com/docs/cli/box.html#capath-certdir) option
    in `vagrant add`; defaults to unset.

-   `add_cert` (string) - Equivalent to setting the
    [`--cert`](https://www.vagrantup.com/docs/cli/box.html#cert-certfile) option in
    `vagrant add`; defaults to unset.

-   `add_clean` (bool) - Equivalent to setting the
    [`--clean`](https://www.vagrantup.com/docs/cli/box.html#clean) flag in
    `vagrant add`; defaults to unset.

-   `add_force` (bool) - Equivalent to setting the
    [`--force`](https://www.vagrantup.com/docs/cli/box.html#force) flag in
    `vagrant add`; defaults to unset.

-   `add_insecure` (bool) - Equivalent to setting the
    [`--insecure`](https://www.vagrantup.com/docs/cli/box.html#insecure) flag in
    `vagrant add`; defaults to unset.

-   `skip_package` (bool) - if true, Packer will not call `vagrant package` to
    package your base box into its own standalone .box file.

-   `output_vagrantfile` (string) - Equivalent to setting the
    [`--vagrantfile`](https://www.vagrantup.com/docs/cli/package.html#vagrantfile-file) option
    in `vagrant package`; defaults to unset

-   `package_include` (string) - Equivalent to setting the
    [`--include`](https://www.vagrantup.com/docs/cli/package.html#include-x-y-z) option
    in `vagrant package`; defaults to unset

## Example

Sample for `hashicorp/precise64` with virtualbox provider.

    {
      "builders": [
        {
          "communicator": "ssh",
          "source_path": "hashicorp/precise64",
          "provider": "virtualbox",
          "add_force": true,
          "type": "vagrant"
        }
      ]
    }

## A note on SSH connections

Currently this builder only works for SSH connections, and automatically fills
in all information needed for the ssh communicator using vagrant's ssh-config.

If you would like to connect via a different username or authentication method
than is produced when you call `vagrant ssh-config`, then you must provide the

`ssh_username` and all other relevant authentication information (e.g.
`ssh_password` or `ssh_private_key_file`)

By providing the `ssh_username`, you're telling Packer not to use the vagrant
ssh config, except for determining the host and port for the virtual machine to
connect to.
