---
description: |
    The shell-local Packer post processor enables users to do some post processing
    after artifacts have been built.
layout: docs
page_title: 'Local Shell - Post-Processors'
sidebar_current: 'docs-post-processors-shell-local'
---

# Local Shell Post Processor

Type: `shell-local`

The local shell post processor executes scripts locally during the post
processing stage. Shell local provides a convenient way to automate executing
some task with packer outputs and variables.

## Basic example

The example below is fully functional.

``` json
{
  "type": "shell-local",
  "inline": ["echo foo"]
}
```

## Configuration Reference

The reference of available configuration options is listed below. The only
required element is either "inline" or "script". Every other option is
optional.

Exactly *one* of the following is required:

-   `command` (string) - This is a single command to execute. It will be
    written to a temporary file and run using the `execute_command` call below.

-   `inline` (array of strings) - This is an array of commands to execute. The
    commands are concatenated by newlines and turned into a single file, so
    they are all executed within the same context. This allows you to change
    directories in one command and use something in the directory in the next
    and so on. Inline scripts are the easiest way to pull off simple tasks
    within the machine.

-   `script` (string) - The path to a script to execute. This path can be
    absolute or relative. If it is relative, it is relative to the working
    directory when Packer is executed.

-   `scripts` (array of strings) - An array of scripts to execute. The scripts
    will be executed in the order specified. Each script is executed in
    isolation, so state such as variables from one script won't carry on to the
    next.

Optional parameters:

-   `environment_vars` (array of strings) - An array of key/value pairs to
    inject prior to the `execute_command`. The format should be `key=value`.
    Packer injects some environmental variables by default into the
    environment, as well, which are covered in the section below.

-   `env_var_format` (string) - When we parse the environment\_vars that you
    provide, this gives us a string template to use in order to make sure that
    we are setting the environment vars correctly. By default on Windows hosts
    this format is `set %s=%s &&` and on Unix, it is `%s='%s'`. You probably
    won't need to change this format, but you can see usage examples for where
    it is necessary below.

-   `execute_command` (array of strings) - The command used to execute the
    script. By default, on \*nix systems this is:

        ["/bin/sh", "-c", "{{.Vars}} {{.Script}}"]

    While on Windows, `execute_command` defaults to:

        ["cmd", "/V", "/C", "{{.Vars}}", "call", "{{.Script}}"]

    This is treated as a [template engine](/docs/templates/engine.html).
    There are two available variables: `Script`, which is the path to the
    script to run, and `Vars`, which is the list of `environment_vars`, if
    configured. If you choose to set this option, make sure that the first
    element in the array is the shell program you want to use (for example,
    "sh" or "/usr/local/bin/zsh" or even "powershell.exe" although anything
    other than a flavor of the shell command language is not explicitly
    supported and may be broken by assumptions made within Packer). It's
    worth noting that if you choose to try to use shell-local for
    Powershell or other Windows commands, the environment variables will
    not be set properly for your environment.

    For backwards compatibility, `execute_command` will accept a string instead
    of an array of strings. If a single string or an array of strings with only
    one element is provided, Packer will replicate past behavior by appending
    your `execute_command` to the array of strings `["sh", "-c"]`. For example,
    if you set `"execute_command": "foo bar"`, the final `execute_command` that
    Packer runs will be \["sh", "-c", "foo bar"\]. If you set
    `"execute_command": ["foo", "bar"]`, the final execute\_command will remain
    `["foo", "bar"]`.

    Again, the above is only provided as a backwards compatibility fix; we
    strongly recommend that you set execute\_command as an array of strings.

-   `inline_shebang` (string) - The
    [shebang](http://en.wikipedia.org/wiki/Shebang_%28Unix%29) value to use
    when running commands specified by `inline`. By default, this is
    `/bin/sh -e`. If you're not using `inline`, then this configuration has no
    effect. **Important:** If you customize this, be sure to include something
    like the `-e` flag, otherwise individual steps failing won't fail the
    provisioner.

-   `keep_input_artifact` (boolean) - Unlike most other post-processors, the
    keep\_input\_artifact option will have no effect for the shell-local
    post-processor. Packer will always retain the input artifact for
    shell-local, since the shell-local post-processor merely passes forward the
    artifact it receives. If your shell-local post-processor produces a file or
    files which you would like to have replace the input artifact, you may
    overwrite the input artifact using the [artifice](./artifice.html)
    post-processor after your shell-local processor has run.

-   `only_on` (array of strings) - This is an array of [runtime operating
    systems](https://golang.org/doc/install/source#environment) where
    `shell-local` will execute. This allows you to execute `shell-local` *only*
    on specific operating systems. By default, shell-local will always run if
    `only_on` is not set."

-   `use_linux_pathing` (bool) - This is only relevant to windows hosts. If you
    are running Packer in a Windows environment with the Windows Subsystem for
    Linux feature enabled, and would like to invoke a bash script rather than
    invoking a Cmd script, you'll need to set this flag to true; it tells
    Packer to use the linux subsystem path for your script rather than the
    Windows path. (e.g. /mnt/c/path/to/your/file instead of
    C:/path/to/your/file). Please see the example below for more guidance on
    how to use this feature. If you are not on a Windows host, or you do not
    intend to use the shell-local post-processor to run a bash script, please
    ignore this option. If you set this flag to true, you still need to provide
    the standard windows path to the script when providing a `script`. This is
    a beta feature.

## Execute Command

To many new users, the `execute_command` is puzzling. However, it provides an
important function: customization of how the command is executed. The most
common use case for this is dealing with **sudo password prompts**. You may
also need to customize this if you use a non-POSIX shell, such as `tcsh` on
FreeBSD.

### The Windows Linux Subsystem

The shell-local post-processor was designed with the idea of allowing you to
run commands in your local operating system's native shell. For Windows, we've
assumed in our defaults that this is Cmd. However, it is possible to run a bash
script as part of the Windows Linux Subsystem from the shell-local
post-processor, by modifying the `execute_command` and the `use_linux_pathing`
options in the post-processor config.

The example below is a fully functional test config.

One limitation of this offering is that "inline" and "command" options are not
available to you; please limit yourself to using the "script" or "scripts"
options instead.

Please note that this feature is still in beta, as the underlying WSL is also
still in beta. There will be some limitations as a result. For example, it will
likely not work unless both Packer and the scripts you want to run are both on
the C drive.

    {
        "builders": [
          {
            "type":         "null",
            "communicator": "none"
          }
        ],
        "provisioners": [
          {
              "type": "shell-local",
              "environment_vars": ["PROVISIONERTEST=ProvisionerTest1"],
              "execute_command": ["bash", "-c", "{{.Vars}} {{.Script}}"],
              "use_linux_pathing": true,
              "scripts": ["C:/Users/me/scripts/example_bash.sh"]
          },
          {
              "type": "shell-local",
              "environment_vars": ["PROVISIONERTEST=ProvisionerTest2"],
              "execute_command": ["bash", "-c", "{{.Vars}} {{.Script}}"],
              "use_linux_pathing": true,
              "script": "C:/Users/me/scripts/example_bash.sh"
          }
      ]
    }

## Default Environmental Variables

In addition to being able to specify custom environmental variables using the
`environment_vars` configuration, the provisioner automatically defines certain
commonly useful environmental variables:

-   `PACKER_BUILD_NAME` is set to the [name of the
    build](/docs/templates/builders.html#named-builds) that Packer is running.
    This is most useful when Packer is making multiple builds and you want to
    distinguish them slightly from a common provisioning script.

-   `PACKER_BUILDER_TYPE` is the type of the builder that was used to create
    the machine that the script is running on. This is useful if you want to
    run only certain parts of the script on systems built with certain
    builders.

## Safely Writing A Script

Whether you use the `inline` option, or pass it a direct `script` or `scripts`,
it is important to understand a few things about how the shell-local
post-processor works to run it safely and easily. This understanding will save
you much time in the process.

### Once Per Builder

The `shell-local` script(s) you pass are run once per builder. That means that
if you have an `amazon-ebs` builder and a `docker` builder, your script will be
run twice. If you have 3 builders, it will run 3 times, once for each builder.

### Interacting with Build Artifacts

In order to interact with build artifacts, you may want to use the [manifest
post-processor](/docs/post-processors/manifest.html). This will write the list
of files produced by a `builder` to a json file after each `builder` is run.

For example, if you wanted to package a file from the file builder into a
tarball, you might write this:

``` json
{
  "builders": [
    {
      "content": "Lorem ipsum dolor sit amet",
      "target": "dummy_artifact",
      "type": "file"
    }
  ],
  "post-processors": [
    [
      {
        "output": "manifest.json",
        "strip_path": true,
        "type": "manifest"
      },
      {
        "inline": [
          "jq \".builds[].files[].name\" manifest.json | xargs tar cfz artifacts.tgz"
        ],
        "type": "shell-local"
      }
    ]
  ]
}
```

This uses the [jq](https://stedolan.github.io/jq/) tool to extract all of the
file names from the manifest file and passes them to tar.

### Always Exit Intentionally

If any post-processor fails, the `packer build` stops and all interim artifacts
are cleaned up.

For a shell script, that means the script **must** exit with a zero code. You
*must* be extra careful to `exit 0` when necessary.

## Usage Examples:

### Windows Host

Example of running a .cmd file on windows:

          {
              "type": "shell-local",
              "environment_vars": ["SHELLLOCALTEST=ShellTest1"],
              "scripts": ["./scripts/test_cmd.cmd"]
          },

Contents of "test\_cmd.cmd":

    echo %SHELLLOCALTEST%

Example of running an inline command on windows: Required customization:
tempfile\_extension

          {
              "type": "shell-local",
              "environment_vars": ["SHELLLOCALTEST=ShellTest2"],
              "tempfile_extension": ".cmd",
              "inline": ["echo %SHELLLOCALTEST%"]
          },

Example of running a bash command on windows using WSL: Required
customizations: use\_linux\_pathing and execute\_command

          {
              "type": "shell-local",
              "environment_vars": ["SHELLLOCALTEST=ShellTest3"],
              "execute_command": ["bash", "-c", "{{.Vars}} {{.Script}}"],
              "use_linux_pathing": true,
              "script": "./scripts/example_bash.sh"
          }

Contents of "example\_bash.sh":

    #!/bin/bash
    echo $SHELLLOCALTEST

Example of running a powershell script on windows: Required customizations:
env\_var\_format and execute\_command

          {
              "type": "shell-local",
              "environment_vars": ["SHELLLOCALTEST=ShellTest4"],
              "execute_command": ["powershell.exe", "{{.Vars}} {{.Script}}"],
              "env_var_format": "$env:%s=\"%s\"; ",
              "script": "./scripts/example_ps.ps1"
          }

Example of running a powershell script on windows as "inline": Required
customizations: env\_var\_format, tempfile\_extension, and execute\_command

          {
              "type": "shell-local",
              "tempfile_extension": ".ps1",
              "environment_vars": ["SHELLLOCALTEST=ShellTest5"],
              "execute_command": ["powershell.exe", "{{.Vars}} {{.Script}}"],
              "env_var_format": "$env:%s=\"%s\"; ",
              "inline": ["write-output $env:SHELLLOCALTEST"]
          }

### Unix Host

Example of running a bash script on unix:

          {
              "type": "shell-local",
              "environment_vars": ["PROVISIONERTEST=ProvisionerTest1"],
              "scripts": ["./scripts/example_bash.sh"]
          }

Example of running a bash "inline" on unix:

          {
              "type": "shell-local",
              "environment_vars": ["PROVISIONERTEST=ProvisionerTest2"],
              "inline": ["echo hello",
                         "echo $PROVISIONERTEST"]
          }

Example of running a python script on unix:

          {
              "type": "shell-local",
              "script": "hello.py",
              "environment_vars": ["HELLO_USER=packeruser"],
              "execute_command": ["/bin/sh", "-c", "{{.Vars}} /usr/local/bin/python {{.Script}}"]
          }

Where "hello.py" contains:

    import os

    print('Hello, %s!' % os.getenv("HELLO_USER"))
