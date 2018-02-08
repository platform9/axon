# Axon
`Axon` describes a pattern and provides a set of tools that allow deployers to
build self-describing, self-configuring container images for their services. It
defines a common 'interface' for a container image that makes it easy for
external deployment tools to initalize and run services and compose collections
of services to do work.

## What is a self-describing container image?
Provisioning tools often need to know detailed information about the services
that run from a container image. For example, the OpenStack nova services use
the same codebase and configuration to run multiple services. In this case it
makes sense to create a single image and expose multiple entrypoints to run
the containers for each service. Deployment orchestration needs to know the
details of these entrypoints and the ports they listen on.

It also helps to have information about external services that need to be made
available to each service. This is especially useful for application setup in
the [decco](https://github.com/platform9/decco) environment where encrypted
tunnels are required for interservice communication.

Rather than forcing deployment tools to have an understanding of the internals
of a container image and its services, the images we build as part
of Axon provide a yaml formatted descriptor as an image LABEL. The descripter
lists available entrypoints, ports, expected locations in the URL space, and
external service connections.

## What is a self-configuring container image?
In the spirit of keeping the knowledge of the services with the container
image, and relieving the external deployment orchestration logic of the need to
understand it, we require each container image to expose an entrypoint,
`init-region`, which takes a standard set of command line parameters that allow
it to access and locate configuration. The init-region invocation is the same
for all service images in an environment, so that a deployment tool doesn't
need to make any distinction between services. To set up an environment for a
list of services, deployment orchestration can simple call `init-region` on
each container image, and start the services.

When we discuss container configuration, we're talking about two things:
* Runtime configuration for the service itself, for example on-disk configuration
  files. This is especially important for services that weren't designed to run
  in containers, and can't read configuration from the environment.
* The runtime environment for a service. Most services require external
  resources or access to other services to do their job and these resources must
  be provisioned before the service can run effectively. It's the job of each
  image's `init-region` to make sure these resources are in place for the
  services that the image provides.
  Examples include:
  * Database schemas and credentials
  * Credentials for messaging, external cloud services, or other facilities.
  * Service registration, for example keystone endpoints.

## Image requirements
To enable the infrastructure described above, we require the following elements
to be implemented for each image.
* When deployed, each container must have access to a shared distributed
  key-value store, like consul, etcd, or zookeeper. So far we've only enable
  consul, but we'd like the system to allow for the possibility of others. This
  store acts as the source of truth for all service configuration parameters.
* Each container image implements an entrypoint called `init-region` in the
  container image's root user PATH. This entrypoint is in charge of
  creating/upgrading the services' external environment. Any results of this
  setup (credentials, addresses etc) must be stored in the key-value store
  described above.
* Each container service has the ability to render its own in-container
  configuration from values in the configuration key-value store. This could be
  as simple as a wrapper script that renders a configuration file from a
  template before starting the service process itself. In many of the
  containers we've created, we run a `watcher` process along side the service,
  that waits for configuration changes, re-renders configuration and restarts
  services.  Tools like [confd](https://github.com/kelseyhightower/confd) and
  [consul-template](https://github.com/hashicorp/consul-template) can be useful
  for this.

## Container Startup Order
External deployment orchestration must often understand the needs of each
service simply to determine the order in which to initialize and start each
container.

For example, say we have a container running rabbitmq, and another container
(call it Consumer) that must have credentials on the rabbit broker to do work.
One way to manage this is to start the rabbit container, and then run
initialization logic for Consumer to create the user. This requires that the
external orchestration logic perform these operations in a strict order: start
rabbit, initialize Consumer, start Consumer. This can get pretty complicated if
there are a lot of services providing and consuming resources like this.

Axon initialization doesn't require actual interaction between services, only
between the service and the configuration store. In the above case, instead of
having Consumer explicitly call rabbitmq to create a user for itself, it simply
adds configuration to the key-value store that expresses the need for that
user. When the rabbit container eventually comes up, its own startup logic, or
a `watcher` running alongside the rabbitmq broker service, can make sure the user
is there.

## Current Status
We've created this model to help us manage customer deployments of OpenStack
and Kubernetes, so the container images we've created so far involve OpenStack
Keystone and a few supporting services. We plan to grow it to include all the
services we currently manage.

It should be possible to use this pattern and the library code here to provision
any set of services.

