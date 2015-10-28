<!--[metadata]>
+++
title = "network rm"
description = "the network rm command description and usage"
keywords = ["network, rm"]
[menu.engine]
parent = "smn_engine_cli"
+++
<![end-metadata]-->

# network rm

    Usage:  docker network rm [OPTIONS] NETWORK

    Deletes a network

      --help=false       Print usage

Removes a network. You cannot remove a network that is in use by 1 or more containers.

```
  $ docker network rm my-network
```
