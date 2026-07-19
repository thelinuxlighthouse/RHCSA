# Chapter 11: Managing Containers with Podman

## Scope notice

As of July 19, 2026, the published RHEL 10 EX200 objectives do **not** list container administration. Older RHCSA versions did. This chapter is retained as valuable RHEL 10 administrator training, but study Chapters 1 through 10 first for the current exam.

## Objectives

After this chapter, you should be able to:

- explain images, containers, registries, pods, namespaces, and rootless operation;
- pull, inspect, run, stop, restart, and remove containers;
- map ports and persist data with volumes or bind mounts;
- apply correct SELinux labeling to mounted content;
- build an image from a Containerfile;
- authenticate to a registry and use fully qualified image names;
- troubleshoot container state, networking, and logs; and
- run a container through systemd with Quadlet.

## 1. Container mental model

An **image** is a read-only packaged file-system and metadata template. A **container** is an image plus a writable layer, isolated processes, and runtime configuration. A **registry** distributes images. A **pod** groups containers that share selected namespaces.

Containers share the host kernel. A VM includes a separate guest kernel. Containers are usually faster and smaller, but they are not simply “small VMs.”

Podman is daemonless: it does not require a central privileged daemon to keep every container running. `conmon` and the OCI runtime help monitor and run individual containers.

## 2. Rootless versus rootful

Rootless Podman runs as an ordinary user and keeps that user's images and containers separate from root's.

Compare:

~~~bash
$ podman info
$ podman ps -a
# podman ps -a
~~~

These can show different stores.

Typical storage:

~~~text
rootful:  /var/lib/containers/storage
rootless: ~/.local/share/containers/storage
~~~

Rootless isolation uses user namespaces and subordinate UID/GID ranges:

~~~bash
$ grep "^$USER:" /etc/subuid /etc/subgid
~~~

Rootless limitations can include restricted device access and inability to bind below the host's unprivileged-port threshold:

~~~bash
$ sysctl net.ipv4.ip_unprivileged_port_start
~~~

Map a high host port such as 8080 instead of weakening the system setting without a real requirement.

## 3. Install and inspect Podman

~~~bash
# dnf install container-tools
$ podman --version
$ podman info
$ podman system df
~~~

Podman configuration commonly lives below `/etc/containers/` and per-user configuration below `~/.config/containers/`.

Use fully qualified image references in repeatable administration:

~~~text
registry/namespace/image:tag
~~~

Example:

~~~text
registry.access.redhat.com/ubi10/ubi:latest
~~~

This avoids ambiguous short-name resolution.

## 4. Images

Pull:

~~~bash
$ podman pull registry.access.redhat.com/ubi10/ubi:latest
~~~

List and inspect:

~~~bash
$ podman images
$ podman image inspect registry.access.redhat.com/ubi10/ubi:latest
$ podman history registry.access.redhat.com/ubi10/ubi:latest
~~~

Search a registry:

~~~bash
$ podman search registry.access.redhat.com/ubi10
~~~

Remove an unused image:

~~~bash
$ podman rmi registry.access.redhat.com/ubi10/ubi:latest
~~~

Do not use force removal until you understand which containers depend on the image.

## 5. Run and manage containers

Run a short command and remove the container afterward:

~~~bash
$ podman run --rm registry.access.redhat.com/ubi10/ubi:latest cat /etc/redhat-release
~~~

Run detached:

~~~bash
$ podman run -d --name demo \
    registry.access.redhat.com/ubi10/ubi:latest \
    sleep infinity
~~~

Inspect state:

~~~bash
$ podman ps
$ podman ps -a
$ podman inspect demo
$ podman top demo
$ podman logs demo
$ podman stats --no-stream demo
~~~

Execute inside:

~~~bash
$ podman exec demo id
$ podman exec -it demo /bin/bash
~~~

Lifecycle:

~~~bash
$ podman stop demo
$ podman start demo
$ podman restart demo
$ podman rm demo
~~~

`podman run` creates a new container. `podman start` starts the same stopped container and preserves its writable layer.

## 6. Port publishing

Generic web example with a lab-supplied image:

~~~bash
$ podman run -d --name web \
    -p 8080:8080 \
    REGISTRY/NAMESPACE/WEB_IMAGE:TAG
~~~

This maps host TCP port 8080 to container port 8080.

Inspect:

~~~bash
$ podman port web
$ ss -lnt
$ curl http://127.0.0.1:8080/
~~~

For remote clients, the host firewall must also allow the host port:

~~~bash
# firewall-cmd --permanent --zone=public --add-port=8080/tcp
# firewall-cmd --reload
~~~

The image's `EXPOSE` metadata documents intended ports but does not publish a host port.

## 7. Persistent storage

Container writable layers are not the right place for important persistent data. Use a named volume or bind mount.

### Named volume

~~~bash
$ podman volume create appdata
$ podman volume inspect appdata
$ podman run -d --name app \
    -v appdata:/var/lib/app \
    REGISTRY/NAMESPACE/APP_IMAGE:TAG
~~~

The volume survives container removal until the volume itself is removed.

### Bind mount with SELinux

~~~bash
$ mkdir -p ~/site
$ printf '%s\n' 'Hello from Podman' > ~/site/index.html
$ podman run -d --name web \
    -p 8080:8080 \
    -v "$HOME/site:/var/www/html:Z" \
    REGISTRY/NAMESPACE/WEB_IMAGE:TAG
~~~

SELinux suffixes:

- `:Z` gives the content a private container label for one container;
- `:z` gives a shared container label for content used by multiple containers.

Use the correct one rather than disabling SELinux.

Check mounts:

~~~bash
$ podman inspect web --format '{{json .Mounts}}'
$ ls -ldZ ~/site
~~~

## 8. Environment, command, and resource settings

~~~bash
$ podman run --rm \
    -e APP_MODE=production \
    -e LOG_LEVEL=info \
    REGISTRY/NAMESPACE/APP_IMAGE:TAG
~~~

Use an environment file for nonsecret configuration:

~~~bash
$ podman run --rm --env-file ./app.env IMAGE
~~~

Do not put passwords in command history or ordinary environment files without considering their exposure.

Resource examples:

~~~bash
$ podman run -d --name limited \
    --memory 512m \
    --cpus 1.5 \
    IMAGE
~~~

Inspect:

~~~bash
$ podman inspect limited --format '{{.HostConfig.Memory}}'
$ podman stats --no-stream limited
~~~

## 9. Registries and authentication

Login:

~~~bash
$ podman login registry.example.com
~~~

Prefer an interactive password prompt or supported secret input. Do not place the password directly in shell history.

Inspect login status when supported:

~~~bash
$ podman login --get-login registry.example.com
~~~

Pull, tag, and push:

~~~bash
$ podman pull registry.example.com/team/app:1.0
$ podman tag local-app:1.0 registry.example.com/team/app:1.0
$ podman push registry.example.com/team/app:1.0
~~~

Logout:

~~~bash
$ podman logout registry.example.com
~~~

Authentication files are per user unless a different path is explicitly configured. Root and an ordinary user do not automatically share registry login.

## 10. Build an image

Example `Containerfile`:

~~~dockerfile
FROM registry.access.redhat.com/ubi10-minimal:latest

COPY healthcheck.sh /usr/local/bin/healthcheck
RUN chmod 755 /usr/local/bin/healthcheck

USER 1001
CMD ["/usr/local/bin/healthcheck"]
~~~

Build:

~~~bash
$ podman build -t localhost/healthcheck:1.0 .
$ podman images localhost/healthcheck
$ podman run --rm localhost/healthcheck:1.0
~~~

Build context matters. `COPY` paths are relative to the context directory, not the Containerfile's absolute location.

A `.containerignore` file prevents unnecessary or sensitive files from entering the build context.

Good image habits:

- pin a meaningful tag when repeatability matters;
- use a trusted base image;
- run as a nonroot user where possible;
- keep secrets out of image layers;
- combine only logically related build steps;
- test the final image, not only the build exit status.

## 11. Pods

Create a pod with a published port:

~~~bash
$ podman pod create --name apppod -p 8080:8080
$ podman run -d --pod apppod --name frontend FRONTEND_IMAGE
$ podman run -d --pod apppod --name backend BACKEND_IMAGE
~~~

Inspect and manage:

~~~bash
$ podman pod ps
$ podman ps --pod
$ podman pod inspect apppod
$ podman pod stop apppod
$ podman pod rm apppod
~~~

Containers in a pod can share the network namespace, so they can commonly reach each other through localhost on their container ports. Do not publish the same pod port separately from multiple member containers.

## 12. Quadlet and systemd

Quadlet is the preferred declarative systemd integration. Older `podman generate systemd` workflows are deprecated.

### Rootless unit

Create:

~~~bash
$ mkdir -p ~/.config/containers/systemd
$ vim ~/.config/containers/systemd/demo.container
~~~

Contents:

~~~ini
[Unit]
Description=Rootless demonstration container

[Container]
Image=registry.access.redhat.com/ubi10/ubi:latest
ContainerName=demo
Exec=sleep infinity

[Service]
Restart=always

[Install]
WantedBy=default.target
~~~

Load and start:

~~~bash
$ systemctl --user daemon-reload
$ systemctl --user start demo.service
$ systemctl --user status demo.service
$ journalctl --user -u demo.service
~~~

The generator turns `demo.container` into `demo.service`. The `[Install]` section provides startup linkage; generated services are not managed like ordinary static files by copying old generated unit output.

To allow the user's manager and service to continue when the user is logged out:

~~~bash
# loginctl enable-linger USERNAME
~~~

Verify:

~~~bash
$ loginctl show-user USERNAME -p Linger
$ systemctl --user list-unit-files | grep demo
~~~

### System-wide rootful unit

Place a `.container` file below `/etc/containers/systemd/`, then:

~~~bash
# systemctl daemon-reload
# systemctl start NAME.service
# systemctl status NAME.service
~~~

Choose rootless or rootful intentionally; their files, storage, ports, and service managers differ.

## 13. Troubleshooting

### Container exits immediately

~~~bash
$ podman ps -a
$ podman logs CONTAINER
$ podman inspect CONTAINER --format '{{.State.ExitCode}} {{.State.Error}}'
~~~

A container normally lives as long as its main process. A successful short command produces an exited container by design.

### Port unavailable

~~~bash
$ podman port CONTAINER
$ ss -lnt
$ sysctl net.ipv4.ip_unprivileged_port_start
$ firewall-cmd --get-active-zones
~~~

Check host-port conflicts, rootless low-port rules, firewall, and whether the application listens inside the container.

### Bind-mount permission denied

~~~bash
$ podman inspect CONTAINER --format '{{json .Mounts}}'
$ ls -ldZ HOST_PATH
$ podman logs CONTAINER
~~~

Check Unix permissions, UID mapping, read-only options, and `:Z` or `:z` labeling.

### Image pull failure

~~~bash
$ getent hosts REGISTRY
$ podman login --get-login REGISTRY
$ podman pull FULLY_QUALIFIED_IMAGE
~~~

Check the exact registry, namespace, tag, credentials, certificate trust, proxy, and DNS.

### Quadlet unit missing

~~~bash
$ systemctl --user daemon-reload
$ systemctl --user list-unit-files | grep NAME
$ journalctl --user -b
~~~

Confirm filename suffix, directory, user, syntax, and image reference.

## 14. Practice lab

1. As an ordinary user, pull the RHEL 10 UBI image.
2. Run a detached container named `demo` whose main process sleeps indefinitely.
3. Bind-mount `~/demo-data` at `/data` with a private SELinux container label.
4. Verify the mount and label.
5. Replace the manually created container with a rootless Quadlet unit.
6. Arrange for the user service manager to continue after logout.

Commands for the manual container:

~~~bash
$ mkdir -p ~/demo-data
$ podman pull registry.access.redhat.com/ubi10/ubi:latest
$ podman run -d --name demo \
    -v "$HOME/demo-data:/data:Z" \
    registry.access.redhat.com/ubi10/ubi:latest \
    sleep infinity
$ podman ps
$ podman inspect demo --format '{{json .Mounts}}'
$ ls -ldZ ~/demo-data
$ podman stop demo
$ podman rm demo
~~~

Then create `~/.config/containers/systemd/demo.container` using the Quadlet pattern, add the volume line:

~~~ini
Volume=%h/demo-data:/data:Z
~~~

Load and verify:

~~~bash
$ systemctl --user daemon-reload
$ systemctl --user start demo.service
$ systemctl --user status demo.service
# loginctl enable-linger USERNAME
$ loginctl show-user USERNAME -p Linger
~~~

## Quick check

- Can you explain why root and a regular user see different container stores?
- Can you distinguish an image from a container?
- Can you explain host port versus container port?
- Can you distinguish a named volume from a bind mount?
- Can you choose between `:Z` and `:z`?
- Can you explain why an exited short-lived container may be correct?
- Can you build from a Containerfile and identify the build context?
- Can you create a rootless Quadlet unit and explain lingering?

## Official references

- [RHEL 10 container documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/building_running_and_managing_containers/index)
- [RHEL 10 introduction to containers](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/building_running_and_managing_containers/introduction-to-containers)
- Local pages: `man podman`, `man podman-run`, `man podman-build`, `man podman-volume`, `man podman-systemd.unit`, and `man loginctl`
