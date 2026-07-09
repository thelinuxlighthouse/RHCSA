# Chapter 11: Managing Containers - Podman

## Learning Objectives

By the end of this chapter, you will be able to:

* Explain core container concepts and how containers differ from virtual machines
* Install and configure Podman on RHEL 10
* Deploy and manage containers using Podman
* Build custom container images using Containerfile
* Manage container storage with volumes, bind mounts, and storage drivers
* Configure container networking including custom networks and port mapping
* Run containers as systemd services for persistent operation
* Use Podman pods to group related containers
* Manage container images from local and remote registries
* Apply security best practices for containerized workloads

## Concepts and Theory

### What Are Containers?

Containers are lightweight, standalone, executable packages that bundle an application with all its dependencies — libraries, configuration files, runtime, and system tools. Unlike virtual machines, containers share the host operating system kernel, which makes them significantly lighter and faster to start.

A container image is a read-only template containing everything needed to run an application. Containers are running instances of images. Images are layered, with each layer representing a change to the filesystem.

### Containers vs. Virtual Machines

Virtual machines include a full guest operating system, a hypervisor layer, and virtualized hardware. Each VM carries the overhead of an entire OS. Containers share the host kernel and only package application-level dependencies. This means containers start in seconds, consume fewer resources, and achieve near-native performance.

### What Is Podman?

Podman is a daemonless, open-source container engine developed by Red Hat for managing OCI-compliant containers and images. It runs on Linux and is the default container tool on RHEL 10. Podman replaces Docker as the primary container runtime on Red Hat systems.

Key characteristics of Podman:

* **Daemonless architecture**: Podman does not require a persistent background daemon. Each Podman command spawns a child process directly, reducing a single point of failure and attack surface.
* **Rootless containers**: Podman supports running containers without root privileges by default, improving security through principle of least privilege.
* **Pod support**: Podman can group multiple containers into a pod, sharing network namespaces and storage, similar to Kubernetes pods.
* **Systemd integration**: Podman can generate systemd unit files to manage containers as system services, enabling automatic restart on boot and failure.
* **Docker compatibility**: Podman uses the same CLI syntax as Docker for most commands, making migration straightforward.

### OCI Compliance

Podman follows the Open Container Initiative (OCI) standards. The OCI defines two key specifications:

* **OCI Image Specification**: Defines how container images are structured, including layered filesystems and metadata.
* **OCI Runtime Specification**: Defines how container images are executed, including security profiles and resource constraints.

This compliance ensures that images built for Docker, Podman, or any OCI-compliant engine are interoperable.

### Container Images and Registries

Container images are stored in registries — centralized repositories for distributing container images. Common registries include:

* **Red Hat Quay**: Red Hat's enterprise container registry.
* **registry.access.redhat.com**: Official Red Hat container catalog for UBI (Universal Base Image) and certified images.
* **Docker Hub**: The default public registry, used when no registry is specified.
* **Quay.io**: A popular public registry hosting community and enterprise images.

Images follow a naming convention: `registry/namespace/image:tag`. If no tag is specified, `latest` is used implicitly.

### Rootless Containers

Rootless containers run under an unprivileged user account. Podman achieves this through:

* **User namespaces**: Map container user IDs to an unused range on the host, isolating the container without requiring root.
* **Subuid/subgid allocation**: The system assigns a range of UIDs and GIDs to the unprivileged user for container use.
* **fuse-overlayfs**: A FUSE-based storage driver that allows rootless users to create overlay filesystems.

Rootless containers cannot bind to privileged ports (below 1024) the value of `net.ipv4.ip_unprivileged_port_start`, cannot modify host network settings, and have restricted access to host devices.

```bash
# Check the current value:
sysctl net.ipv4.ip_unprivileged_port_start
```

Changing this kernel setting requires administrative privileges and
should be considered carefully because it affects unprivileged port
binding on the host.

## How It Works Internally

### Namespaces

Linux namespaces isolate container resources from the host and from other containers. Podman uses the following namespaces:

* **PID namespace**: Isolates process IDs, so a container sees only its own processes.
* **Network namespace**: Provides an isolated network stack with its own interfaces, routes, and firewall rules.
* **Mount namespace**: Isolates the filesystem view, so a container sees only its own mount points.
* **UTS namespace**: Isolates hostname and domain name.
* **IPC namespace**: Isolates inter-process communication resources.
* **User namespace**: Maps UIDs and GIDs between the container and the host.

### Control Groups (cgroups)

Control groups limit and account for resource usage. Podman uses cgroups v2 on RHEL 10 to constrain container CPU, memory, I/O, and device access. cgroups v2 uses a unified hierarchy managed by the `systemd` cgroup controller, integrating seamlessly with systemd resource management.

### Overlay Filesystem

Podman uses an overlay filesystem to build container images efficiently. Each image layer is stored once on disk. When a container runs, a writable layer is placed on top of the read-only image layers. Changes made inside the container are written to this writable layer, leaving the image layers untouched.

In rootless mode, Podman uses `fuse-overlayfs`, a FUSE-based implementation that does not require kernel privileges. In root mode, it uses the kernel-native `overlay` driver.

### Storage Driver Architecture

Podman stores container data under `/var/lib/containers` for root operations and `$HOME/.local/share/containers` for rootless operations. The storage configuration is defined in:

* `/etc/containers/storage.conf` — system-wide storage configuration
* `$HOME/.config/containers/storage.conf` — per-user storage configuration

The default storage driver on RHEL 10 is `overlay`, which leverages the kernel's overlayfs support for efficient layer management.

### Podman Process Model

When you run `podman run`, Podman forks a child process that manages the container lifecycle. This process uses `conmon` (container monitor) as a wrapper to handle I/O, monitor container health, and manage cleanup. If the Podman parent process dies, `conmon` ensures the container is properly cleaned up.

This daemonless model means there is no `podman.service` running in the background. Each command is self-contained, which improves reliability and simplifies debugging.

## Commands and Administration Tasks

### Installing Podman

Podman is installed by default on RHEL 10. Verify the installation:

```bash
podman --version
```

If Podman is not installed, install it using dnf:

```bash
sudo dnf install -y podman
```

Additional useful packages include:

```bash
sudo dnf install -y buildah skopeo containers-common
```

* `buildah` — alternative tool for building container images
* `skopeo` — tool for inspecting and copying container images
* `containers-common` — shared configuration and policies

### Managing Container Images

#### Pulling Images

Download an image from a registry:

```bash
podman pull docker.io/library/nginx:latest
```

Pull all tags of an image:

```bash
podman pull --all docker.io/library/nginx
```

#### Listing Images

```bash
podman images
```

Show all images, including intermediate layers:

```bash
podman images -a
```

#### Removing Images

Remove a single image:

```bash
podman rmi docker.io/library/nginx:latest
```

Force removal if containers are using the image:

```bash
podman rmi -f docker.io/library/nginx
```

Remove all unused images:

```bash
podman image prune -a
```

#### Inspecting Images

View detailed metadata about an image:

```bash
podman inspect docker.io/library/nginx:latest
```

View the history of layers in an image:

```bash
podman history docker.io/library/nginx:latest
```

### Running Containers

#### Basic Container Launch

```bash
podman run -d --name webserver -p 8080:80 docker.io/library/nginx:latest
```

Flag breakdown:

* `-d` — run in detached (background) mode
* `--name webserver` — assign a human-readable name
* `-p 8080:80` — map host port 8080 to container port 80
* `docker.io/library/nginx:latest` — image to run

#### Interactive Containers

Run a container interactively with a terminal:

```bash
podman run -it --name debug-shell docker.io/library/ubi10:latest /bin/bash
```

Flags:

* `-i` — keep STDIN open
* `-t` — allocate a pseudo-TTY

#### One-Shot Commands

Execute a single command in a container and exit:

```bash
podman run --rm docker.io/library/ubi10:latest cat /etc/os-release
```

The `--rm` flag automatically removes the container after it exits.

#### Environment Variables

Pass environment variables to a container:

```bash
podman run -d --name myapp -e APP_ENV=production -e LOG_LEVEL=info myapp:latest
```

Load environment variables from a file:

```bash
podman run -d --name myapp --env-file /path/to/envfile myapp:latest
```

### Managing Running Containers

#### Listing Containers

List running containers:

```bash
podman ps
```

List all containers, including stopped ones:

```bash
podman ps -a
```

Show container IDs only:

```bash
podman ps -q
```

#### Starting, Stopping, and Restarting

```bash
podman start webserver
podman stop webserver
podman restart webserver
```

Stop all running containers:

```bash
podman stop -a
```

#### Executing Commands in Running Containers

```bash
podman exec -it webserver /bin/bash
```

Run a single command without a shell:

```bash
podman exec webserver nginx -v
```

#### Viewing Container Logs

```bash
podman logs webserver
```

Follow logs in real time:

```bash
podman logs -f webserver
```

Show last 50 lines:

```bash
podman logs --tail 50 webserver
```

#### Inspecting Containers

```bash
podman inspect webserver
```

Get only the IP address:

```bash
podman inspect --format '{{.NetworkSettings.IPAddress}}' webserver
```

#### Removing Containers

Remove a stopped container:

```bash
podman rm webserver
```

Force remove a running container:

```bash
podman rm -f webserver
```

Remove all stopped containers:

```bash
podman container prune
```

### Container Resource Limits

Limit memory usage:

```bash
podman run -d --name memory-limited --memory=512m myapp:latest
```

Limit CPU usage:

```bash
podman run -d --name cpu-limited --cpus=1.5 myapp:latest
```

Set both limits:

```bash
podman run -d --name limited --memory=256m --cpus=1 myapp:latest
```

### Building Custom Images

Podman uses a `Containerfile` (or `Dockerfile`) to define image build instructions.

Create a Containerfile:

```dockerfile
FROM docker.io/library/ubi10:latest
RUN dnf install -y nginx && dnf clean all
COPY index.html /var/www/html/index.html
EXPOSE 80
CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```

Build the image:

```bash
podman build -t my-nginx:1.0 .
```

Tag and push to a registry:

```bash
podman tag my-nginx:1.0 registry.example.com/my-nginx:1.0
podman push registry.example.com/my-nginx:1.0
```

### Podman Pods

A pod groups multiple containers that share the same network namespace, allowing them to communicate via `localhost`.

Create a pod:

```bash
podman pod create --name webstack -p 8080:80
```

Run containers in the pod:

```bash
podman run -d --pod webstack --name nginx docker.io/library/nginx:latest
podman run -d --pod webstack --name php docker.io/library/php:apache
```

List pods:

```bash
podman pod ps
```

Remove a pod and its containers:

```bash
podman pod rm -f webstack
```

## Configuration Examples

### Podman System Configuration

The main system-wide configuration file is `/etc/containers/containers.conf`:

```toml
[engine]
stop_timeout = 30
num_threads = 4
events_logger = "file"

[containers]
tz = "local"
pids_limit = 200
```

### Network Configuration

Podman network configuration resides in `/etc/containers/networks.conf`. The default network is `podman` (formerly `bridge`).

Create a custom network:

```bash
podman network create --driver bridge --subnet 10.10.0.0/24 --gateway 10.10.0.1 appnet
```

Connect a container to the custom network:

```bash
podman run -d --name app1 --network appnet myapp:latest
```

### Registry Configuration

Configure registries in `/etc/containers/registries.conf`:

```toml
[[registry]]
prefix = "docker.io"
location = "docker.io"

[[registry]]
prefix = "registry.access.redhat.com"
location = "registry.access.redhat.com"

unqualified-search-registries = ["docker.io", "registry.access.redhat.com"]
```

### Storage Configuration

Edit `/etc/containers/storage.conf` to adjust storage settings:

```toml
[storage]
driver = "overlay"
graphroot = "/var/lib/containers/storage"
runroot = "/run/containers/storage"

[storage.options.overlay]
mount_program = "/usr/bin/fuse-overlayfs"
```

### Rootless Container Setup

Enable rootless containers for a user:

```bash
sudo sh -c 'echo "wdigo 100000 65536" >> /etc/subuid'
sudo sh -c 'echo "wdigo 100000 65536" >> /etc/subgid'
```

Verify the user has a UID range allocated:

```bash
grep wdigo /etc/subuid
```

Start the rootless CNI network:

```bash
podman system migrate
```

### Systemd Service Generation

Generate a systemd unit file for a running container:

```bash
podman generate systemd --new --name webserver --files --restart-policy on-failure
```

This creates a `.service` file in the current directory. Move it to the appropriate systemd directory:

```bash
sudo mv container-webserver.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now container-webserver.service
```

For rootless containers, generate user-level units:

```bash
podman generate systemd --new --name webserver --files --user --restart-policy on-failure
mv container-webserver.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now container-webserver.service
```

## Real-World Examples

### Example 1: Deploying a Web Server

Deploy an Nginx web server with a custom HTML page, persistent configuration, and automatic restart on boot.

Create the directory structure:

```bash
sudo mkdir -p /srv/webserver/html
sudo mkdir -p /srv/webserver/config
```

Create a custom index page:

```bash
sudo tee /srv/webserver/html/index.html > /dev/null << 'EOF'
<!DOCTYPE html>
<html>
<head><title>RHCSA Web Server</title></head>
<body>
<h1>RHCSA EX200 - Containerized Web Server</h1>
<p>This web server is running inside a Podman container on RHEL 10.</p>
</body>
</html>
EOF
```

Run the container with bind mounts:

```bash
sudo podman run -d \
  --name rhcsa-web \
  --restart=unless-stopped \
  -p 8080:80 \
  -v /srv/webserver/html:/usr/share/nginx/html:ro \
  -v /srv/webserver/config:/etc/nginx/conf.d:ro \
  docker.io/library/nginx:latest
```

Verify the service is running:

```bash
curl http://localhost:8080
```

Generate a systemd unit for persistence:

```bash
sudo podman generate systemd --new --name rhcsa-web --files --restart-policy on-failure
sudo mv container-rhcsa-web.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable container-rhcsa-web.service
```

### Example 2: Multi-Container Application with a Pod

Deploy a web application with a frontend and a backend database in the same pod.

Create the pod:

```bash
sudo podman pod create --name appstack -p 3000:3000 -p 5432:5432
```

Deploy the database:

```bash
sudo podman run -d \
  --pod appstack \
  --name db \
  -e POSTGRES_PASSWORD=securepass123 \
  -e POSTGRES_DB=appdb \
  -v pgdata:/var/lib/postgresql/data \
  docker.io/library/postgres:16
```

Deploy the application:

```bash
sudo podman run -d \
  --pod appstack \
  --name app \
  -e DATABASE_HOST=localhost \
  -e DATABASE_PORT=5432 \
  -e DATABASE_NAME=appdb \
  myapp:latest
```

Because both containers share the pod's network namespace, the application connects to the database using `localhost`.

### Example 3: Building a Custom Image

Build a custom image for a Python application.

Create a project directory:

```bash
mkdir -p ~/python-app
cd ~/python-app
```

Create the application file:

```bash
tee app.py > /dev/null << 'EOF'
from http.server import HTTPServer, BaseHTTPRequestHandler
import datetime

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.end_headers()
        self.wfile.write(f"Hello from container. Time: {datetime.datetime.now()}\n".encode())

if __name__ == "__main__":
    server = HTTPServer(("0.0.0.0", 8080), Handler)
    server.serve_forever()
EOF
```

Create the Containerfile:

```bash
tee Containerfile > /dev/null << 'EOF'
FROM docker.io/library/ubi10/ubi-python-311:latest
WORKDIR /app
COPY app.py /app/app.py
EXPOSE 8080
CMD ["python", "app.py"]
EOF
```

Build and run:

```bash
podman build -t python-app:1.0 .
podman run -d --name python-svc -p 8080:8080 python-app:1.0
```

### Example 4: Rootless Container Deployment

Deploy a container as a regular user without root privileges.

Ensure the user has subuid/subgid ranges:

```bash
grep $USER /etc/subuid
```

If no output, request allocation from the system administrator:

```bash
sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 $USER
```

Run a container rootlessly:

```bash
podman run -d --name my-rootless-app -p 8888:80 docker.io/library/nginx:latest
```

Verify the container is running under the user:

```bash
podman ps
```

Generate a user-level systemd unit:

```bash
podman generate systemd --new --name my-rootless-app --files --user --restart-policy always
mv container-my-rootless-app.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now container-my-rootless-app.service
```

## Troubleshooting

### Container Fails to Start

Check the container status and logs:

```bash
podman ps -a
podman logs <container-name>
podman inspect <container-name>
```

Look for exit codes in the inspect output under `State.ExitCode`. Common exit codes:

* `0` — container exited normally
* `1` — application error inside the container
* `125` — Podman-level error (image not found, port conflict, etc.)
* `137` — container was killed (OOM or SIGKILL)

### Port Already in Use

If a container fails with a port binding error:

```bash
ss -tlnp | grep :8080
```

Kill the conflicting process or use a different host port:

```bash
podman run -d -p 9090:80 nginx:latest
```

### Image Pull Fails

Check network connectivity and DNS resolution:

```bash
ping registry.access.redhat.com
nslookup registry.access.redhat.com
```

Verify registry configuration:

```bash
cat /etc/containers/registries.conf
```

Check for authentication issues:

```bash
podman login registry.access.redhat.com
```

### Storage Issues

Check available disk space:

```bash
df -h /var/lib/containers
df -h /run/containers
```

Inspect storage driver status:

```bash
podman info | grep -A 5 "store"
```

Clean up unused resources:

```bash
podman system prune -a --volumes

#  -a  --  Expands pruning to all unused images, not images actively required by containers.
```

### Rootless Container Failures

If rootless containers fail to start, verify user namespace setup:

```bash
cat /etc/subuid | grep $USER
cat /etc/subgid | grep $USER
```

Check the fuse-overlayfs driver:

```bash
which fuse-overlayfs
```

If missing, install it:

```bash
sudo dnf install -y fuse-overlayfs
```

Verify the storage driver for rootless mode:

```bash
podman info | grep graphDriverName
```

### Container Cannot Reach Network

Check the container's network configuration:

```bash
podman inspect --format '{{.NetworkSettings.Networks}}' <container-name>
```

Verify the default network exists:

```bash
podman network ls
```

Restart the container's network:

```bash
podman network disconnect podman <container-name>
podman network connect podman <container-name>
```

### SELinux Blocking Container Operations

Check audit logs for SELinux denials:

```bash
sudo ausearch -m avc -ts recent
sudo grep denied /var/log/audit/audit.log
```

If SELinux is blocking bind mounts, ensure the volume has the correct label. Podman handles labeling automatically when you use the `:z` or `:Z` suffix on bind mounts:

```bash
podman run -d -v /data:/container-data:z myapp:latest
```

### Systemd Service for Container Fails

Check the systemd service status:

```bash
systemctl status container-<name>.service
journalctl -u container-<name>.service
```

Verify the unit file references the correct image and options:

```bash
cat /etc/systemd/system/container-<name>.service
```

Reload systemd and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart container-<name>.service
```

## Verification Procedures

### Verify Podman Installation and Version

```bash
podman --version
podman info
```

### Verify Container Is Running

```bash
podman ps --filter name=<container-name>
podman inspect --format '{{.State.Running}}' <container-name>
```

### Verify Port Mapping

```bash
podman port <container-name>
ss -tlnp | grep <port>
```

### Verify Container Logs

```bash
podman logs <container-name>
```

### Verify Image Exists

```bash
podman images | grep <image-name>
podman inspect <image-name>
```

### Verify Network Configuration

```bash
podman network ls
podman network inspect <network-name>
podman inspect --format '{{.NetworkSettings.IPAddress}}' <container-name>
```

### Verify Volume Mounts

```bash
podman inspect --format '{{.Mounts}}' <container-name>
podman volume ls
```

### Verify Systemd Integration

```bash
systemctl is-enabled container-<name>.service
systemctl status container-<name>.service
```

### Verify Resource Limits

```bash
podman inspect --format '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}' <container-name>
```

### Verify Rootless Container Configuration

```bash
id -u
podman info | grep Rootless
cat /etc/subuid | grep $USER
```

## RHCSA Exam Notes

### What to Expect on the Exam

The RHCSA EX200 exam includes tasks related to container management with Podman. You will likely be asked to:

* Pull images from specified registries
* Run containers with specific options (port mapping, environment variables, resource limits)
* Build a custom image using a Containerfile
* Generate systemd unit files to persist containers across reboots
* Manage container networking
* Use volumes and bind mounts for persistent data
* Work with rootless containers

### Key Commands to Memorize

| Task | Command |
|------|---------|
| Pull image | `podman pull <image>` |
| Run container | `podman run -d --name <name> <image>` |
| List containers | `podman ps -a` |
| Stop container | `podman stop <name>` |
| Remove container | `podman rm <name>` |
| View logs | `podman logs <name>` |
| Execute command | `podman exec -it <name> <cmd>` |
| Build image | `podman build -t <name> .` |
| Inspect container | `podman inspect <name>` |
| Create network | `podman network create <name>` |
| Generate systemd unit | `podman generate systemd --new --name <name> --files` |
| Create pod | `podman pod create --name <name>` |

### Common Exam Scenarios

**Scenario 1**: You are asked to deploy a container that serves a specific web page on a given port. You must pull the correct image, run the container with port mapping, bind mount the content, and ensure it starts on boot.

**Scenario 2**: You are asked to create a Containerfile that installs a package and runs a service. You must write the Containerfile, build the image, and verify the container runs correctly.

**Scenario 3**: You are asked to generate a systemd service file for an existing container so that it restarts after a reboot. Use `podman generate systemd` with the correct flags.

**Scenario 4**: You are asked to configure a container with resource limits. Use `--memory` and `--cpus` flags.

### Exam Tips

* Always verify your work. After completing a container task, run `podman ps` and `podman logs` to confirm the container is healthy.
* Use `--rm` for temporary test containers to avoid cluttering the environment.
* When generating systemd units, always run `systemctl daemon-reload` afterward.
* For rootless container tasks, confirm that subuid/subgid ranges are configured.
* Pay attention to the exact image names and tags specified in exam questions. Using the wrong tag may result in a different application version.
* When asked to persist container data, use named volumes (`-v volumename:/path`) or bind mounts (`-v /host/path:/container/path`).
* The `Containerfile` name is preferred over `Dockerfile` in Podman, though both are supported. Use `Containerfile` in the exam.
* Remember that `podman generate systemd` creates files in the current directory. You must move them to the correct systemd directory and reload the daemon.

### Time Management

Container tasks typically take 5-10 minutes each. Do not spend excessive time debugging image pulls — if an image fails to pull, check the registry URL and your network configuration, then move on.

## Chapter Summary

Podman is the default container engine on RHEL 10, providing a daemonless, secure alternative to Docker. Containers offer lightweight, isolated environments for running applications without the overhead of virtual machines. Podman supports rootless containers, pod grouping, systemd integration, and full OCI compliance.

Key concepts covered in this chapter include container images and registries, running and managing containers, building custom images with Containerfile, managing storage through volumes and bind mounts, configuring container networks, and deploying containers as systemd services for persistent operation.

Understanding Podman is essential for the RHCSA EX200 exam, where you will be expected to deploy, manage, and persist containerized services on RHEL 10 systems.

## Quick Reference

### Essential Podman Commands

```bash
# Image management
podman pull <image>
podman images
podman rmi <image>
podman inspect <image>

# Container lifecycle
podman run -d --name <name> <image>
podman start <name>
podman stop <name>
podman restart <name>
podman rm <name>
podman ps -a

# Container interaction
podman exec -it <name> /bin/bash
podman logs <name>
podman inspect <name>

# Networking
podman network create <name>
podman network ls
podman network inspect <name>

# Storage
podman volume create <name>
podman volume ls
podman volume inspect <name>

# Building images
podman build -t <name>:<tag> .

# Systemd integration
podman generate systemd --new --name <name> --files --restart-policy on-failure

# Pods
podman pod create --name <name>
podman pod ps
podman pod rm <name>

# Cleanup
podman system prune -a
```

### Common Run Flags

| Flag | Purpose |
|------|---------|
| `-d` | Detached mode |
| `-it` | Interactive terminal |
| `-p host:container` | Port mapping |
| `-e KEY=VALUE` | Environment variable |
| `--env-file <file>` | Environment file |
| `-v src:dest` | Volume or bind mount |
| `--name <name>` | Container name |
| `--rm` | Remove on exit |
| `--memory=<size>` | Memory limit |
| `--cpus=<number>` | CPU limit |
| `--network <name>` | Network to join |
| `--pod <name>` | Run in pod |
| `--restart=unless-stopped` | Restart policy |

### Configuration Files

| File | Purpose |
|------|---------|
| `/etc/containers/containers.conf` | System-wide container config |
| `/etc/containers/storage.conf` | Storage driver config |
| `/etc/containers/registries.conf` | Registry configuration |
| `/etc/containers/policy.json` | Image signing policy |
| `$HOME/.config/containers/` | Per-user configuration |

### Storage Locations

| Mode | Image Storage | Runtime Storage |
|------|---------------|-----------------|
| Root | `/var/lib/containers/storage` | `/run/containers/storage` |
| Rootless | `~/.local/share/containers/storage` | `~/.local/share/containers/tmp` |

## Review Questions

1. What is the primary architectural difference between Podman and Docker?
2. What does the `-d` flag do when running `podman run`?
3. How do you map host port 8080 to container port 80?
4. What command generates a systemd unit file for a running container?
5. What is a Podman pod and what does it share among its containers?
6. Which storage driver does Podman use by default on RHEL 10?
7. How do you run a container without root privileges?
8. What file is used to define custom container images in Podman?
9. What command lists all container networks?
10. How do you limit a container to 512 MB of memory?
11. What is the purpose of the `--rm` flag?
12. Where are rootless container images stored?
13. What command executes a shell inside a running container?
14. What is the OCI standard and why does it matter for Podman?
15. How do you view the logs of a container named `webapp`?

## Answers

1. Podman is daemonless — it does not run a persistent background daemon. Each command spawns its own child process, whereas Docker relies on a central `dockerd` daemon.
2. The `-d` flag runs the container in detached mode, meaning it runs in the background and returns control to the terminal immediately.
3. Use the `-p 8080:80` flag with `podman run`.
4. `podman generate systemd --new --name <container-name> --files`
5. A Podman pod is a group of containers that share the same network namespace, allowing them to communicate via `localhost`. They also share the same IP address and ports.
6. The `overlay` storage driver is used by default on RHEL 10.
7. Run Podman as a regular user without `sudo`. The user must have subuid/subgid ranges allocated in `/etc/subuid` and `/etc/subgid`.
8. A `Containerfile` (or `Dockerfile`) defines the instructions for building a custom container image.
9. `podman network ls`
10. Use the `--memory=512m` flag with `podman run`.
11. The `--rm` flag automatically removes the container when it exits, preventing stopped containers from accumulating.
12. Rootless container images are stored in `~/.local/share/containers/storage`.
13. `podman exec -it <container-name> /bin/bash`
14. OCI (Open Container Initiative) defines standards for container images and runtime behavior. OCI compliance ensures that images built for any OCI-compliant engine (Docker, Podman, etc.) are interchangeable.
15. `podman logs webapp`
