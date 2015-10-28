<!--[metadata]>
+++
title = "Network plugins"
description = "Manage networks with external plugins"
keywords = ["Examples, Usage, plugins, docker, documentation, user guide"]
[menu.main]
parent = "mn_extend"
weight=-1
+++
<![end-metadata]-->

# Network plugins

Docker network plugins enable Docker deployments to be extended to support
a wide range of networking technologies, such as VXLAN, IPVLAN, MACVLAN or
something completely different!

## Using network plugins

First install your plugin according to the instructions obtained from the
plugin developer.

Once installed, you may use the new network driver in the `docker network
create` command. For example,

    $ docker network create -d weave mynet

Some network plugins are listed in [plugins](plugins.md)

The `mynet` network is now owned by `weave`, so subsequent commands
referring to that network will be sent to the plugin,

    $ docker run --net=mynet busybox top

# Write a network plugin

Network plugins implement the [Docker plugin API](https://docs.docker.com/extend/plugin_api/)
and the network plugin protocol

## Network plugin protocol

The network driver protocol, in addition to the plugin activation call, is
documented as part of libnetwork:
[https://github.com/docker/libnetwork/blob/master/docs/remote.md](https://github.com/docker/libnetwork/blob/master/docs/remote.md).

****** MARY - WE SHOULD VENDOR THIS IN SOMEHOW, LIBNETWORK DOCS AREN'T HUGO COMPATIBLE

