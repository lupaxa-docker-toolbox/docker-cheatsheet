<p align="center">
  <a href="https://github.com/lupaxa-docker-toolbox">
    <img src="https://raw.githubusercontent.com/the-lupaxa-project/brand-assets/master/logos/organisations/docker-toolbox/readme-logo.png" alt="The Lupaxa Project: Docker Toolbox" />
  </a>
</p>

<h1 align="center">Docker Cheatsheet</h1>

A current command sheet for Docker Engine, Compose, BuildKit, and the Hub. Commands use the Docker CLI that ships with Engine and Desktop. Compose is `docker compose`.

Official references:

- [Learn Docker](https://docs.docker.com/learn/)
- [Install Engine](https://docs.docker.com/engine/install/)
- [Docker Desktop](https://docs.docker.com/desktop/)
- [CLI reference](https://docs.docker.com/reference/cli/docker/)
- [Output formatting](https://docs.docker.com/engine/cli/formatting/)
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [BuildKit](https://docs.docker.com/build/)
- [Compose file](https://docs.docker.com/reference/compose-file/)
- [Networking](https://docs.docker.com/engine/network/)
- [Volumes](https://docs.docker.com/engine/storage/volumes/)
- [Logging](https://docs.docker.com/engine/logging/configure/)
- [Daemon configuration](https://docs.docker.com/engine/daemon/)
- [Swarm](https://docs.docker.com/engine/swarm/)

## Contents

- [What Docker Is](#what-docker-is)
- [Install Docker](#install-docker)
- [Client and Daemon](#client-and-daemon)
- [Containers](#containers)
- [Inspect and Logs](#inspect-and-logs)
- [Resource Limits](#resource-limits)
- [Images](#images)
- [Build an Image](#build-an-image)
- [Networks](#networks)
- [Registry](#registry)
- [Dockerfile](#dockerfile)
- [Image Layers](#image-layers)
- [Compose](#compose)
- [Volumes](#volumes)
- [Published Ports](#published-ports)
- [Logging](#logging)
- [Swarm](#swarm)
- [Practices](#practices)
- [Security](#security)
- [Cleanup](#cleanup)

## What Docker Is

A container is a process on the host, isolated by namespaces and cgroups, with its own filesystem built from image layers. An image is a read-only stack of those layers. A registry stores images. The Hub is the default registry.

A tag is a movable name. A digest (`sha256:…`) identifies one image and does not move when the tag is updated.

## Install Docker

On macOS and Windows, install [Docker Desktop](https://docs.docker.com/desktop/). On Linux, install Engine from the packages for that distribution: [Install Docker Engine](https://docs.docker.com/engine/install/).

```bash
docker version
docker info
```

> **Note:** Docker Machine and the old Docker Toolbox virtual machine are retired. Use Desktop or Engine.

## Client and Daemon

The client talks to a daemon. `docker context` selects which daemon.

```bash
docker context ls
docker context show
docker context use default
docker context create remote --docker "host=ssh://user@host"
```

`DOCKER_HOST` overrides the context for one command. `docker --debug` prints client debug logs.

On Linux, Engine reads `/etc/docker/daemon.json`. Restart the daemon after changing it. Desktop keeps its own settings UI. A small daemon file:

```json
{
  "log-driver": "local",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true,
  "default-address-pools": [
    {
      "base": "10.90.0.0/16",
      "size": 24
    }
  ]
}
```

`live-restore` keeps containers running across a daemon restart. `default-address-pools` avoids the default `172.17.0.0/16` bridge when that range clashes with a VPN.

## Containers

Create and start. `--rm` removes the container when it exits. `-d` detaches. `-it` attaches a terminal. `--pull` is `missing`, `always`, or `never`.

```bash
docker run --rm -it ubuntu:24.04 bash
docker run -d --name web --pull missing -p 8080:80 nginx:1.27
docker create --name web nginx:1.27
```

### Lifecycle

```bash
docker ps
docker ps -a
docker start web
docker stop -t 20 web
docker restart web
docker pause web
docker unpause web
docker kill --signal SIGTERM web
docker wait web
docker rm web
docker rm -f web
docker rename web web-old
```

`stop` sends the stop signal (SIGTERM unless the image sets another), waits `--timeout` seconds, then kills the container. `wait` prints the exit code. `pause` freezes processes with the freezer cgroup. It does not stop them.

`docker debug web` opens a shell with extra tools. It is a Desktop plugin. `exec` is the Engine command.

### Run Options

```bash
docker run -d --name web \
  --hostname web \
  --restart unless-stopped \
  -e APP_ENV=prod \
  --env-file ./web.env \
  -w /usr/share/nginx/html \
  --user 65534:65534 \
  --entrypoint nginx \
  -l app=demo \
  nginx:1.27 -g 'daemon off;'
```

| Flag                           | Effect                                                                     |
| ------------------------------ | -------------------------------------------------------------------------- |
| `--restart`                    | `no`, `on-failure[:N]`, `always`, or `unless-stopped`.                     |
| `-e` / `--env-file`            | Environment variables. A file is `KEY=value` lines.                        |
| `-w`                           | Working directory.                                                         |
| `--user`                       | `uid`, `uid:gid`, or a name from the image.                                |
| `--entrypoint`                 | Replaces the image `ENTRYPOINT`. The image `CMD` is dropped.               |
| `--init`                       | Runs a tiny init as PID 1 so zombies are reaped and signals are forwarded. |
| `-l`                           | Adds a label. Filters and Compose use labels.                              |
| `--add-host name:host-gateway` | Maps a name to the host, as seen from the container.                       |

Arguments after the image replace `CMD`.

### Inside a Running Container

```bash
docker exec -it web sh
docker exec -u 0 web cat /etc/nginx/nginx.conf
docker logs web
docker logs -f --tail 50 --timestamps web
docker logs --since 10m --until 2m web
docker attach web
docker top web
docker stats web
docker stats --no-stream web
docker diff web
docker port web
```

`exec` starts another process in the same namespaces. `attach` connects to the original process. Detach from `attach` with `Ctrl-p Ctrl-q`.

### Copy Files

```bash
docker cp web:/etc/nginx/nginx.conf ./nginx.conf
docker cp ./index.html web:/usr/share/nginx/html/index.html
```

`cp` follows the path as it is in the container. A copy into a running container is not an image layer.

### Health Check

```bash
docker run -d --name web \
  --health-cmd 'nginx -t' \
  --health-interval 30s \
  --health-timeout 3s \
  --health-retries 3 \
  --health-start-period 10s \
  nginx:1.27
docker inspect --format '{{.State.Health.Status}}' web
```

Status is `starting`, `healthy`, or `unhealthy`. Exit 0 from the command is healthy. Exit 1 is unhealthy.

## Inspect and Logs

`inspect` prints JSON. `--format` is a Go template.

```bash
docker inspect web
docker inspect --format '{{.State.Status}} {{.State.ExitCode}}' web
docker inspect --format '{{.State.OOMKilled}}' web
docker inspect --format '{{.HostConfig.RestartPolicy.Name}}' web
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}} {{end}}' web
docker image inspect --format '{{.Os}}/{{.Architecture}}' nginx:1.27
docker image inspect --format '{{index .RepoDigests 0}}' nginx:1.27
```

The old `.NetworkSettings.IPAddress` field is the default bridge only. On a user-defined network, read `.NetworkSettings.Networks`.

### Formats

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker ps --format json
docker image ls --digests
docker stats --no-stream --format 'table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}'
```

### Filters

```bash
docker ps -a --filter status=exited
docker ps --filter name=web --filter label=app=demo
docker ps -aq
docker image ls --filter dangling=true
docker image ls --filter reference='nginx:*'
docker volume ls --filter dangling=true
docker network ls --filter driver=bridge
```

`-q` prints ids only, which is what you pass to another command.

### Events

```bash
docker events --since 10m
docker events --filter container=web --filter event=die
```

## Resource Limits

```bash
docker run -d --name web \
  --cpus 1 \
  --cpuset-cpus 0,1 \
  --cpu-shares 512 \
  --memory 256m \
  --memory-swap 256m \
  --memory-reservation 128m \
  --pids-limit 100 \
  --ulimit nofile=1024:2048 \
  nginx:1.27
```

`--cpus` is a hard CPU cap. `--cpu-shares` is a relative weight (the default is 1024) and only matters when the host is busy.
`--memory` is a hard memory cap. `--memory-swap` is memory plus swap. Set it equal to `--memory` to disable swap.
`--pids-limit` caps the number of processes, which limits fork bombs.

Change limits on a running container:

```bash
docker update --cpus 1 --memory 256m web
```

### Devices

```bash
docker run --rm --gpus all nvidia/cuda:12.6.0-base-ubuntu24.04 nvidia-smi
docker run --rm --device /dev/net/tun ubuntu:24.04 true
```

`--gpus` needs the NVIDIA container runtime. `--device` passes one host device through.

## Images

```bash
docker pull nginx:1.27
docker image ls
docker image inspect nginx:1.27
docker history nginx:1.27
docker tag nginx:1.27 lupaxa/web:1.27
docker image rm nginx:1.27
docker search nginx --limit 5
```

`search` queries the Hub and returns a short list. The Hub website is the fuller catalogue.

`docker save` and `docker load` move an image, including its history. `docker export` writes a container's filesystem and drops the history. `docker import` turns that tarball into a single-layer image.

```bash
docker save nginx:1.27 -o nginx.tar
docker load -i nginx.tar
docker export web -o web.tar
docker import web.tar lupaxa/web:flat
```

`commit` snapshots a container into an image. There is no Dockerfile for that image. Prefer a build.

```bash
docker commit web lupaxa/web:dirty
```

### Digests

```bash
docker pull nginx:1.27
docker image ls --digests nginx
docker pull nginx@sha256:<digest>
docker buildx imagetools inspect nginx:1.27
```

Pin a deployed image by digest when the tag must not move under you. Use the digest `image ls --digests` printed.

## Build an Image

Current Docker builds with BuildKit through Buildx.

```bash
docker build -t lupaxa/web:1 .
docker build --pull --no-cache -t lupaxa/web:1 .
docker build --target runtime -t lupaxa/web:1 .
docker build --build-arg VERSION=1 -t lupaxa/web:1 .
docker init
```

`docker init` writes starter Docker files for a project. `--target` stops at one stage. `--pull` refreshes the base image.

### Secrets and Cache

A secret mount is available during that `RUN` and is not copied into the layer. A cache mount keeps a directory between builds.

```dockerfile
RUN --mount=type=secret,id=npm,target=/root/.npmrc \
    npm ci
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

```bash
docker build --secret id=npm,src=.npmrc -t lupaxa/web:1 .
docker build --ssh default -t lupaxa/web:1 .
```

The matching Dockerfile line for the agent is `RUN --mount=type=ssh …`. See [Build secrets](https://docs.docker.com/build/building/secrets/).

Cache export lets another machine reuse the build:

```bash
docker build --cache-to type=local,dest=./cache -t lupaxa/web:1 .
docker build --cache-from type=local,src=./cache -t lupaxa/web:1 .
```

### Multi-platform Images

```bash
docker buildx ls
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t lupaxa/web:1 \
  --push \
  .
docker buildx bake
```

A multi-platform build pushes a manifest list, or loads one platform with `--load`. `bake` builds the targets named in a Bake file.

## Networks

Put containers on a user-defined network. They reach each other by container name. The default bridge does not provide that name lookup.

```bash
docker network create app
docker network ls
docker network inspect app
docker run -d --name web --network app --network-alias frontend nginx:1.27
docker network connect app web
docker network disconnect app web
docker network rm app
```

```bash
docker network create \
  --driver bridge \
  --subnet 10.90.0.0/24 \
  --gateway 10.90.0.1 \
  app
docker network create --internal private
```

`--internal` has no route out of the Docker host. An alias is an extra DNS name on that network.

### Network Modes

| Mode             | Meaning                                                              |
| ---------------- | -------------------------------------------------------------------- |
| `bridge`         | Private network on the host. This is the default.                    |
| `host`           | Share the host network namespace. This is Linux Engine, not Desktop. |
| `none`           | No network interfaces except loopback inside the container.          |
| `container:name` | Share another container's network namespace.                         |
| `overlay`        | Swarm network across nodes.                                          |

```bash
docker run --rm --network host ubuntu:24.04 true
docker run --rm --network none ubuntu:24.04 true
docker run --rm --network container:web ubuntu:24.04 true
```

> **Note:** `--link` is legacy. Use a user-defined network.

## Registry

Names are `registry/namespace/repository:tag`. A name with no registry goes to Docker Hub.

```bash
docker login
docker tag lupaxa/web:1 lupaxa/web:1.0.0
docker push lupaxa/web:1.0.0
docker pull lupaxa/web:1.0.0
docker logout
```

### Private Registry

```bash
docker login "$REGISTRY"
docker tag lupaxa/web:1 "$REGISTRY/lupaxa/web:1"
docker push "$REGISTRY/lupaxa/web:1"
```

The daemon must trust the registry's certificate, or the host must be listed under `insecure-registries` in the daemon config. Prefer a real certificate.

## Dockerfile

```dockerfile
FROM ubuntu:24.04 AS build
WORKDIR /src
COPY . .
RUN make

FROM ubuntu:24.04
RUN useradd --create-home --uid 10001 app
WORKDIR /app
COPY --from=build --chmod=755 /src/out/app /app/app
USER app
EXPOSE 8080
STOPSIGNAL SIGTERM
HEALTHCHECK --interval=30s --timeout=3s \
  CMD ["/app/app", "health"]
ENTRYPOINT ["/app/app"]
CMD ["serve"]
```

| Instruction   | Role                                                                |
| ------------- | ------------------------------------------------------------------- |
| `FROM`        | Base image. A later `FROM` starts a new stage.                      |
| `RUN`         | Run a command and commit a layer.                                   |
| `COPY`        | Copy files from the build context.                                  |
| `ADD`         | Copy files, and also unpack archives or fetch a URL. Prefer `COPY`. |
| `WORKDIR`     | Set the working directory.                                          |
| `ENV`         | Set an environment variable in the image.                           |
| `ARG`         | Build-time variable. It is not kept unless you assign it to `ENV`.  |
| `EXPOSE`      | Document the port the process listens on. It does not publish it.   |
| `USER`        | User for the following commands and the running container.          |
| `VOLUME`      | Mark a path for data that should outlive the container.             |
| `ENTRYPOINT`  | Fixed command.                                                      |
| `CMD`         | Default arguments, or the command when there is no `ENTRYPOINT`.    |
| `HEALTHCHECK` | Command the daemon runs to decide whether the container is healthy. |
| `LABEL`       | Metadata on the image.                                              |
| `SHELL`       | Shell used by shell-form `RUN`, `CMD`, and `ENTRYPOINT`.            |
| `STOPSIGNAL`  | Signal `docker stop` sends. The default is SIGTERM.                 |
| `ONBUILD`     | Trigger that runs in a child image, not in this one.                |

Shell form (`RUN apt-get update`) runs through a shell. Exec form (`CMD ["serve"]`) does not, and it receives signals directly.

A heredoc is still one layer:

```dockerfile
RUN <<'EOF'
set -eux
apt-get update
apt-get install -y --no-install-recommends ca-certificates
rm -rf /var/lib/apt/lists/*
EOF
```

`COPY --from` takes files from an earlier stage or from another image. `COPY --chmod` sets permissions without a following `RUN`.

Automatic build arguments include `TARGETOS`, `TARGETARCH`, and `TARGETPLATFORM`. Use them when one Dockerfile builds more than one architecture.

## Image Layers

Each `RUN`, `COPY`, and `ADD` adds a layer. Docker reuses a layer when the instruction and everything before it are unchanged. Put files that change often, such as source, after dependencies.

A `.dockerignore` next to the Dockerfile keeps secrets, `.git`, and build output out of the context:

```text
.git
.venv
**/node_modules
*.log
```

## Compose

Compose runs a group of containers from `compose.yaml`. The project name prefixes the network and volumes. `compose.override.yaml` in the same directory is merged automatically.

```yaml
services:
  web:
    image: nginx:1.27
    ports:
      - "127.0.0.1:8080:80"
    restart: unless-stopped
    profiles:
      - frontend
    depends_on:
      api:
        condition: service_healthy
  api:
    build:
      context: ./api
      target: runtime
    env_file:
      - ./api.env
    healthcheck:
      test: ["CMD", "/app/app", "health"]
      interval: 30s
      timeout: 3s
      retries: 3
    develop:
      watch:
        - action: rebuild
          path: ./api
    volumes:
      - api-data:/var/lib/api
volumes:
  api-data:
```

```bash
docker compose config
docker compose up -d
docker compose --profile frontend up -d
docker compose -f compose.yaml -f compose.prod.yaml up -d
docker compose up -d --scale api=3
docker compose ps
docker compose logs -f --since 10m api
docker compose exec api sh
docker compose run --rm api pytest
docker compose build
docker compose pull
docker compose watch
docker compose ls
docker compose stop
docker compose down
docker compose down -v
```

`config` prints the file after interpolation and merges. `run` starts a one-off container for a service.
`down` removes the containers and the project network. `-v` also removes named volumes declared in the file.
`watch` follows the `develop.watch` rules. See [Compose Watch](https://docs.docker.com/compose/how-tos/file-watch/).

A service with a profile starts only when that profile is enabled. In this file, `docker compose up -d` starts `api` and skips `web`.

`${VAR}` in the file comes from the shell or from a `.env` file next to the compose file. `env_file` is the container environment, which is a different file.

## Volumes

A named volume is Docker-managed storage. A bind mount uses a host path. An anonymous volume gets a random name. Prefer a named volume for database files.

```bash
docker volume create data
docker volume ls
docker volume inspect data
docker volume rm data
docker run -d --name db -v data:/var/lib/postgresql/data postgres:17
docker run --rm -v "$PWD":/work:ro -w /work ubuntu:24.04 ls
```

The explicit form:

```bash
docker run --rm \
  --mount type=volume,source=data,destination=/var/lib/postgresql/data \
  postgres:17
docker run --rm \
  --mount type=bind,source="$PWD",destination=/work,readonly \
  ubuntu:24.04 ls /work
```

`:ro` and `readonly` make the mount read-only inside the container.

### Tmpfs

```bash
docker run --rm --tmpfs /tmp:rw,nosuid,nodev,size=64m ubuntu:24.04 df -h /tmp
```

A tmpfs lives in memory and disappears when the container stops. Use it for scratch files when the root filesystem is read-only.

> **Note:** A data-only container is an old pattern. Use a named volume.

## Published Ports

`-p host:container` publishes a port. An address in front binds that host address only. `-P` publishes every `EXPOSE`d port on a random host port.

```bash
docker run -d --name web -p 8080:80 nginx:1.27
docker run -d --name web -p 127.0.0.1:8080:80 nginx:1.27
docker run -d --name web -p 8080:80/udp nginx:1.27
docker run -d --name web -P nginx:1.27
docker port web
```

Publishing a port makes it reachable on the host's network. `EXPOSE` alone does not. `127.0.0.1` keeps the port on the loopback interface.

## Logging

The default driver is `json-file`. `local` is the faster on-disk driver. Other drivers include `journald`, `syslog`, `fluentd`, and `none`.

```bash
docker run -d --name web \
  --log-driver local \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  nginx:1.27
docker inspect --format '{{.HostConfig.LogConfig.Type}}' web
```

`docker logs` works for `json-file` and `local`. It does not work for drivers that ship logs elsewhere. Set the daemon default with `log-driver` in `daemon.json`.

## Swarm

Swarm is Engine's built-in orchestrator. New clusters are usually Kubernetes. These commands manage a Swarm you already run.

```bash
docker swarm init
docker swarm join-token worker
docker swarm join --token TOKEN host:2377
docker node ls
docker service create --name web --replicas 3 -p 8080:80 nginx:1.27
docker service ls
docker service ps web
docker service scale web=5
docker service logs web
docker service update --image nginx:1.28 web
docker service rollback web
docker service rm web
docker stack deploy -c compose.yaml app
docker stack ls
docker stack services app
docker stack rm app
docker swarm leave --force
```

A service is reconciled. If a task dies, Swarm starts another. `stack deploy` submits a Compose file to the swarm. `leave --force` is how a manager leaves.

Secrets and configs in Swarm are separate objects:

```bash
docker secret create db-password ./db-password.txt
docker config create nginx-conf ./nginx.conf
```

## Practices

- Pin the image tag you intend to ship. Pin the digest when the tag must not move.
- One main process per container. Let the restart policy or the orchestrator restart it.
- Build in one stage and copy the result into a smaller final image.
- Run as a non-root `USER`.
- Keep the Dockerfile context small with `.dockerignore`.
- Sort package lists and remove the package index in the same `RUN` as the install.
- Pass credentials with `--secret` or `--ssh`. Do not `COPY` them into a layer.
- Set a log size. An unbounded `json-file` log fills the disk.
- Publish a port on `127.0.0.1` when only the host should reach it.

## Security

The Docker daemon is root-equivalent. Anyone in the `docker` group can start a privileged container and control the host.

- Do not mount the Docker socket into a container.
- Do not pass `--use-api-socket` unless that container is meant to control Docker.
- Avoid `--privileged`.
- Drop capabilities, then add back only what the process needs.
- Set a process limit.
- Make the root filesystem read-only and mount a tmpfs where writes are required.
- Prefer [rootless mode](https://docs.docker.com/engine/security/rootless/) on a shared machine.
- On a multi-user Linux host, `userns-remap` in `daemon.json` maps container root to an unprivileged host user.

```bash
docker run -d --name web \
  --read-only \
  --tmpfs /tmp \
  --user 65534:65534 \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges \
  --pids-limit 100 \
  nginx:1.27
```

`no-new-privileges` stops the process raising privileges through a setuid binary. `--cap-drop ALL` starts from an empty set. Add back a capability only when the process fails without it.

Optional image scan, when the Scout plugin is installed:

```bash
docker scout quickview nginx:1.27
docker scout cves nginx:1.27
```

## Cleanup

```bash
docker system df
docker container prune
docker image prune
docker volume prune
docker network prune
docker builder prune
docker system prune
```

`docker system prune` removes stopped containers, unused networks, and dangling images.
Add `--volumes` only when unused named volumes should go too.
Add `-a` to `docker image prune`, or use `docker system prune -a`, to remove every image not used by a container.

Narrow a prune with a filter:

```bash
docker image prune -a --filter until=24h
docker builder prune --filter until=168h
docker container prune --filter label=app=demo
```

Stop and remove containers you named, rather than every container on the daemon:

```bash
docker stop web
docker rm web
```

<a href="https://github.com/the-lupaxa-project">
  <img src="https://raw.githubusercontent.com/the-lupaxa-project/brand-assets/master/logos/components/footer-for-child-orgs.svg" alt="The Lupaxa Project Footer" width="100%" />
</a>
