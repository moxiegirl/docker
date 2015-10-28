<!--[metadata]>
+++
title = "stop"
description = "The stop command description and usage"
keywords = ["stop, SIGKILL, SIGTERM"]
[menu.engine]
parent = "smn_engine_cli"
+++
<![end-metadata]-->

# stop

    Usage: docker stop [OPTIONS] CONTAINER [CONTAINER...]

    Stop a container by sending SIGTERM and then SIGKILL after a
    grace period

      --help=false       Print usage
      -t, --time=10      Seconds to wait for stop before killing it

The main process inside the container will receive `SIGTERM`, and after a grace
period, `SIGKILL`.
