# Configuring DNS
<a name="dns"></a>

How can Docker supply each container with a hostname and DNS configuration, without having to build a custom image with the hostname written inside?  Its trick is to overlay three crucial `/etc` files inside the container with virtual files where it can write fresh information.  You can see this by running `mount` inside a container:

```
$$ mount
...
/dev/disk/by-uuid/1fec...ebdf on /etc/hostname type ext4 ...
/dev/disk/by-uuid/1fec...ebdf on /etc/hosts type ext4 ...
/dev/disk/by-uuid/1fec...ebdf on /etc/resolv.conf type ext4 ...
...
```

This arrangement allows Docker to do clever things like keep `resolv.conf` up to date across all containers when the host machine receives new configuration over DHCP later.  The exact details of how Docker maintains these files inside the container can change from one Docker version to the next, so you should leave the files themselves alone and use the following Docker options instead.

Four different options affect container domain name services.
- `-h HOSTNAME` or `--hostname=HOSTNAME` -- sets the hostname by which

  the container knows itself.  This is written into `/etc/hostname`,

  into `/etc/hosts` as the name of the container's host-facing IP

  address, and is the name that `/bin/bash` inside the container will

  display inside its prompt.  But the hostname is not easy to see from

  outside the container.  It will not appear in `docker ps` nor in the

  `/etc/hosts` file of any other container.

- `--link=CONTAINER_NAME_or_ID:ALIAS` -- using this option as you `run` a

  container gives the new container's `/etc/hosts` an extra entry

  named `ALIAS` that points to the IP address of the container identified by

  `CONTAINER_NAME_or_ID`.  This lets processes inside the new container

  connect to the hostname `ALIAS` without having to know its IP.  The

  `--link=` option is discussed in more detail below, in the section

  [Communication between containers](#between-containers). Because

  Docker may assign a different IP address to the linked containers

  on restart, Docker updates the `ALIAS` entry in the `/etc/hosts` file

  of the recipient containers.

- `--dns=IP_ADDRESS...` -- sets the IP addresses added as `server`

  lines to the container's `/etc/resolv.conf` file.  Processes in the

  container, when confronted with a hostname not in `/etc/hosts`, will

  connect to these IP addresses on port 53 looking for name resolution

  services.

- `--dns-search=DOMAIN...` -- sets the domain names that are searched

  when a bare unqualified hostname is used inside of the container, by

  writing `search` lines into the container's `/etc/resolv.conf`.

  When a container process attempts to access `host` and the search

  domain `example.com` is set, for instance, the DNS logic will not

  only look up `host` but also `host.example.com`.

  Use `--dns-search=.` if you don't wish to set the search domain.

- `--dns-opt=OPTION...` -- sets the options used by DNS resolvers

  by writing an `options` line into the container's `/etc/resolv.conf`.

  See documentation for `resolv.conf` for a list of valid options.

Regarding DNS settings, in the absence of the `--dns=IP_ADDRESS...`, `--dns-search=DOMAIN...`, or `--dns-opt=OPTION...` options, Docker makes each container's `/etc/resolv.conf` look like the `/etc/resolv.conf` of the host machine (where the `docker` daemon runs).  When creating the container's `/etc/resolv.conf`, the daemon filters out all localhost IP address `nameserver` entries from the host's original file.

Filtering is necessary because all localhost addresses on the host are unreachable from the container's network.  After this filtering, if there  are no more `nameserver` entries left in the container's `/etc/resolv.conf` file, the daemon adds public Google DNS nameservers (8.8.8.8 and 8.8.4.4) to the container's DNS configuration.  If IPv6 is enabled on the daemon, the public IPv6 Google DNS nameservers will also be added (2001:4860:4860::8888 and 2001:4860:4860::8844).

> **Note**: If you need access to a host's localhost resolver, you must modify your DNS service on the host to listen on a non-localhost address that is reachable from within the container.

You might wonder what happens when the host machine's `/etc/resolv.conf` file changes.  The `docker` daemon has a file change notifier active which will watch for changes to the host DNS configuration.

> **Note**: The file change notifier relies on the Linux kernel's inotify feature. Because this feature is currently incompatible with the overlay filesystem  driver, a Docker daemon using "overlay" will not be able to take advantage of the `/etc/resolv.conf` auto-update feature.

When the host file changes, all stopped containers which have a matching `resolv.conf` to the host will be updated immediately to this newest host configuration.  Containers which are running when the host configuration changes will need to stop and start to pick up the host changes due to lack of a facility to ensure atomic writes of the `resolv.conf` file while the container is running. If the container's `resolv.conf` has been edited since it was started with the default configuration, no replacement will be attempted as it would overwrite the changes performed by the container. If the options (`--dns`, `--dns-search`, or `--dns-opt`) have been used to modify the default host configuration, then the replacement with an updated host's `/etc/resolv.conf` will not happen as well.

> **Note**: For containers which were created prior to the implementation of the `/etc/resolv.conf` update feature in Docker 1.5.0: those containers will **not** receive updates when the host `resolv.conf` file changes. Only containers created with Docker 1.5.0 and above will utilize this auto-update feature.
