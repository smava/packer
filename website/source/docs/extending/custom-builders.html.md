---
description: |
    It is possible to write custom builders using the Packer plugin interface, and
    this page documents how to do that.
layout: docs
page_title: 'Custom Builders - Extending'
sidebar_current: 'docs-extending-custom-builders'
---

# Custom Builders

Packer Builders are the components of Packer responsible for creating a
machine, bringing it to a point where it can be provisioned, and then turning
that provisioned machine into some sort of machine image. Several builders are
officially distributed with Packer itself, such as the AMI builder, the VMware
builder, etc. However, it is possible to write custom builders using the Packer
plugin interface, and this page documents how to do that.

Prior to reading this page, it is assumed you have read the page on [plugin
development basics](/docs/extending/plugins.html).

~&gt; **Warning!** This is an advanced topic. If you're new to Packer, we
recommend getting a bit more comfortable before you dive into writing plugins.

## The Interface

The interface that must be implemented for a builder is the `packer.Builder`
interface. It is reproduced below for reference. The actual interface in the
source code contains some basic documentation as well explaining what each
method should do.

``` go
type Builder interface {
  Prepare(...interface{}) error
  Run(context.Context, ui Ui, hook Hook) (Artifact, error)
}
```

### The "Prepare" Method

The `Prepare` method for each builder is called prior to any runs with the
configuration that was given in the template. This is passed in as an array of
`interface{}` types, but is generally `map[string]interface{}`. The prepare
method is responsible for translating this configuration into an internal
structure, validating it, and returning any errors.

For multiple parameters, they should be merged together into the final
configuration, with later parameters overwriting any previous configuration.
The exact semantics of the merge are left to the builder author.

For decoding the `interface{}` into a meaningful structure, the
[mapstructure](https://github.com/mitchellh/mapstructure) library is
recommended. Mapstructure will take an `interface{}` and decode it into an
arbitrarily complex struct. If there are any errors, it generates very human
friendly errors that can be returned directly from the prepare method.

While it is not actively enforced, **no side effects** should occur from
running the `Prepare` method. Specifically, don't create files, don't launch
virtual machines, etc. Prepare's purpose is solely to configure the builder and
validate the configuration.

In addition to normal configuration, Packer will inject a
`map[string]interface{}` with a key of `packer.DebugConfigKey` set to boolean
`true` if debug mode is enabled for the build. If this is set to true, then the
builder should enable a debug mode which assists builder developers and
advanced users to introspect what is going on during a build. During debug
builds, parallelism is strictly disabled, so it is safe to request input from
stdin and so on.

### The "Run" Method

`Run` is where all the interesting stuff happens. Run is executed, often in
parallel for multiple builders, to actually build the machine, provision it,
and create the resulting machine image, which is returned as an implementation
of the `packer.Artifact` interface.

The `Run` method takes three parameters. These are all very useful. The
`packer.Ui` object is used to send output to the console. `packer.Hook` is used
to execute hooks, which are covered in more detail in the hook section below.
And `packer.Cache` is used to store files between multiple Packer runs, and is
covered in more detail in the cache section below.

Because builder runs are typically a complex set of many steps, the
[multistep](https://github.com/hashicorp/packer/blob/master/helper/multistep)
helper is recommended to bring order to the complexity. Multistep is a library
which allows you to separate your logic into multiple distinct "steps" and
string them together. It fully supports cancellation mid-step and so on. Please
check it out, it is how the built-in builders are all implemented.

Finally, as a result of `Run`, an implementation of `packer.Artifact` should be
returned. More details on creating a `packer.Artifact` are covered in the
artifact section below. If something goes wrong during the build, an error can
be returned, as well. Note that it is perfectly fine to produce no artifact and
no error, although this is rare.

### Cancellation

The `Run` method is often run in parallel.

#### With the "Cancel" Method ( up until packer 1.3 )

The `Cancel` method can be called at any time and requests cancellation of any
builder run in progress. This method should block until the run actually stops.
Not that the Cancel method will no longer be called since packer 1.4.0.

#### Context cancellation ( from packer 1.4 )

The `<-ctx.Done()` can unblock at any time and signifies request for
cancellation of any builder run in progress.

Cancels are most commonly triggered by external interrupts, such as the user
pressing `Ctrl-C`. Packer will only exit once all the builders clean up, so it
is important that you architect your builder in a way that it is quick to
respond to these cancellations and clean up after itself.

## Creating an Artifact

The `Run` method is expected to return an implementation of the
`packer.Artifact` interface. Each builder must create their own implementation.
The interface has ample documentation to help you get started.

The only part of an artifact that may be confusing is the `BuilderId` method.
This method must return an absolutely unique ID for the builder. In general, I
follow the practice of making the ID contain my GitHub username and then the
platform it is building for. For example, the builder ID of the VMware builder
is "hashicorp.vmware" or something similar.

Post-processors use the builder ID value in order to make some assumptions
about the artifact results, so it is important it never changes.

Other than the builder ID, the rest should be self-explanatory by reading the
[packer.Artifact interface
documentation](https://github.com/hashicorp/packer/blob/master/packer/artifact.go).

## Provisioning

Packer has built-in support for provisioning, but the moment when provisioning
runs must be invoked by the builder itself, since only the builder knows when
the machine is running and ready for communication.

When the machine is ready to be provisioned, run the `packer.HookProvision`
hook, making sure the communicator is not nil, since this is required for
provisioners. An example of calling the hook is shown below:

``` go
hook.Run(context.Context, packer.HookProvision, ui, comm, nil)
```

At this point, Packer will run the provisioners and no additional work is
necessary.

-&gt; **Note:** Hooks are still undergoing thought around their general design
and will likely change in a future version. They aren't fully "baked" yet, so
they aren't documented here other than to tell you how to hook in provisioners.
