---
title: 'Offline, Multistage Python Dockerfile'
published: true
description: Multistage and Offline Installation Dockerfile for Python
tags: 'docker,python,devops,debian'
id: 2530646
date: '2025-05-26T21:00:16Z'
---

There Was a need to reduce a python application docker image size, also had to have offline installation either in the PYPI packages installation or in APT update and installing packages.

Lets get to the Optimization Line by Line.

## Docker build `buildx` `syntax`

- First of all make sure docker uses the `docker buildx` as its default `docker build`.

[BuildKit][docker-buildkit-getting-start]

> If you have installed Docker Desktop, you don't need to enable BuildKit. If you are running a version of Docker Engine version earlier than 23.0, you can enable BuildKit either by setting an environment variable, or by making BuildKit the default setting in the daemon configuration.

```bash
DOCKER_BUILDKIT=1 docker build --file /path/to/dockerfile -t docker_image_name:tag
```

- `# syntax=docker/dockerfile:1.4` for the [heredoc in Dockerfile][heredoc-dockerfile].

```docker
# syntax=docker/dockerfile:1.4 # Required for heredocs [3, 4]
```

## Project Directory Tree

```tree
├── main.py
├── requirements.txt
└── src
    ├── log.py
    └── prometheus.py
```

## Multistage Dockerfile

as mentioned before, at first provisioning the `base` stage to be used in the next `build` and `runtime` stages.

### `base` stage

- base image

```docker
ARG JFROG=jfrog.example.com

FROM ${JFROG}/docker/python:3.13-slim AS base
```

- Change the default SHELL
  - A safe way with custom shell with `pipefail` and `errexit` options, its very useful in the Heredoc in the Debian Private repo setup section.

```docker
SHELL ["/bin/bash", "-c", "-o", "pipefail", "-o", "errexit"]
```

- Environments, see [Python Env Variables in Dockerfile][python-env-var-in-dockerfile]

```docker
ARG JFROG=jfrog.example.com

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=on \
    PIP_TIMEOUT=60 \
    PIP_INDEX_URL=https://${JFROG}/artifactory/api/pypi/python/simple/
```

- Private Debian Repository (Offline Installation)

   Used [`heredoc`][heredoc-dockerfile] in Docker to change the base image apt sources to update and install packages from private Debian repository. heredoc needs the [dockerfile syntax][custom-dockerfile-syntax] mentioned before.</br>
   If the structure of the Debian repo is different in a private repo , please change the `URIs`.

[DEB822 format (apt .sources files)][deb822-style-format]

```docker
# Using DEB822 format (.sources files) - for newer systems
RUN <<EOF

CODENAME=$(grep VERSION_CODENAME /etc/os-release | cut -d'=' -f2)
DISTRO=$(grep '^ID=' /etc/os-release | cut -d'=' -f2)

cat > /etc/apt/sources.list.d/debian.sources <<SOURCE_FILE_CONTENT
Types: deb
URIs: https://${JFROG}/artifactory/debian/debian/
Suites: ${CODENAME} ${CODENAME}-updates
Components: main
Trusted: true

Types: deb
URIs: https://${JFROG}/artifactory/debian/debian-security/
Suites: ${CODENAME}-security
Components: main
Trusted: true
SOURCE_FILE_CONTENT
EOF
```

- Install Shared and common packages in all stages.

  - In the package installation there is no need to install recommended packages to reduces the image size.
  - After installation, for the sake of size image there is need to remove packages downloads. </br></br>

```docker
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    && rm -rf /var/lib/apt/lists/*
```

### `build` stage

- Use the prepared `base` image as `build` image

```docker
FROM base AS build
```

- There was no need for build specific packages in all stages, so just install them in `build` stage.

```docker
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential && \
    rm -rf /var/lib/apt/lists/*
```

- Install requirements

  - Change directory to `app`.
  - Create virtualenv, in the `runtime` stage, [the `virtualenv` will be copied][copy-virtual-env] in image.
  - use the [cache mount][cache-mount] for faster build.
  - For the sake of image size install requirements with [disabling pip cache][disabling-caching] with `--no-cache-dir` flag. </br></br>

```docker
WORKDIR /app

RUN python -m venv .venv
ENV PATH="/app/.venv/bin:$PATH"

COPY requirements.txt .

RUN --mount=type=cache,target=/root/.cache/pip \
pip --timeout 100 install --no-cache-dir -r requirements.txt
```

### `runtime` stage

- Use the prepared `base` image as `build` image

```docker
FROM base AS build

WORKDIR /app
```

- Security best practices

  - Create `group` and `user` to leverage the kubernetes `runAsUser`, `runAsGroup` and `fsGroup` [`securityContext`][pod-security-context] </br></br>

```docker
RUN addgroup --gid 1001 --system nonroot && \
    adduser --no-create-home --shell /bin/false \
    --disabled-password --uid 1001 --system --group nonroot

USER nonroot:nonroot
```

- VirtualEnv

  - Add the `/app/.venv/bin` into `PATH`.
  - [Copy the `virtualenv`][copy-virtual-env] from build stage. </br></br>

```docker
ENV VIRTUAL_ENV=/app/.venv \
    PATH="/app/.venv/bin:$PATH"

COPY --from=build --chown=nonroot:nonroot /app/.venv /app/.venv
```

- Copy `src` directory.

```docker
COPY --chown=nonroot:nonroot src /app/src
COPY --chown=nonroot:nonroot main.py .
```

- `CMD` to run container from image.

```docker
CMD ["python", "/app/main.py"]
```

## Before And After The optimization

</br></br>

### Before The Optimization

</br></br>

The Dockerfile was:

```docker
FROM jfrog.example.com/docker/python:latest

WORKDIR /app
ADD src/ .

RUN pip config set global.index-url https://jfrog.example.com/artifactory/api/pypi/python/simple/ &&  \
    pip --timeout 100 install -r requirements.txt

CMD ["python","-u","main.py"]
```

After build its size was `1.02GB`.

### Final Dockerfile After Optimization

</br></br>

After all Optimization and multistage Dockerfile its size reduced to `242MB`.

```docker
# syntax=docker/dockerfile:1.4 # Required for heredocs [3, 4]

ARG PYTHON_VERSION=3.12.3
ARG JFROG=jfrog.example.com

FROM ${JFROG}/docker/python:${PYTHON_VERSION}-slim AS base
SHELL ["/bin/bash", "-c", "-o", "pipefail", "-o", "errexit"]

ARG JFROG=jfrog.example.com

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=on \
    PIP_TIMEOUT=60 \
    PIP_INDEX_URL=https://${JFROG}/artifactory/api/pypi/python/simple/

# Using DEB822 format (.sources files) - for newer systems
RUN <<EOF

CODENAME=$(grep VERSION_CODENAME /etc/os-release | cut -d'=' -f2)
# DISTRO=$(grep '^ID=' /etc/os-release | cut -d'=' -f2)

cat > /etc/apt/sources.list.d/debian.sources <<SOURCE_FILE_CONTENT
Types: deb
URIs: https://${JFROG}/artifactory/debian/debian/
Suites: ${CODENAME} ${CODENAME}-updates
Components: main
Trusted: true

Types: deb
URIs: https://${JFROG}/artifactory/debian/debian-security/
Suites: ${CODENAME}-security
Components: main
Trusted: true
SOURCE_FILE_CONTENT
EOF

# securely copy .netrc using BuildKit secrets
RUN --mount=type=secret,id=netrc,target=/root/.netrc \
    apt-get update && apt-get install --no-install-recommends --no-install-suggests -y \
      ca-certificates \
      gnupg \
      curl \
    && apt-get clean \
    && apt-get remove --purge --auto-remove -y \
    && rm -rf /var/lib/apt/lists/*

FROM base AS build

# securely copy .netrc using BuildKit secrets
RUN --mount=type=secret,id=netrc,target=/root/.netrc \
    apt-get update && apt-get install --no-install-recommends --no-install-suggests -y \
      build-essential \
    && apt-get clean \
    && apt-get remove --purge --auto-remove -y \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

RUN python -m venv .venv
ENV PATH="/app/.venv/bin:$PATH"

COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install --timeout 100 --no-cache-dir --upgrade pip && \
    pip install --timeout 100 --no-cache-dir -r requirements.txt

FROM base AS runtime

WORKDIR /app

RUN addgroup --gid 1001 --system nonroot && \
    adduser --no-create-home --shell /bin/false \
    --disabled-password --uid 1001 --system --group nonroot

USER nonroot:nonroot

ENV VIRTUAL_ENV=/app/.venv \
    PATH="/app/.venv/bin:$PATH"

COPY --from=build --chown=nonroot:nonroot /app/.venv /app/.venv
COPY --chown=nonroot:nonroot src /app/src
COPY --chown=nonroot:nonroot main.py .

CMD ["python", "/app/main.py"]

```

## Notice

After a second thought, realized there is no need for dockerfile syntax version 1.4 to manipulate the `apt` sources, it could be done with `sed`.

The new dockerfile syntax was fun to learn, so i keep this guide as it is.

The [Dockerfile][dockerfile-no-new-syntax] with no new syntax and manipulating the `apt` sources via `sed`.

## Updates

- Mon Jul 14 2025
  - Add Credentials for Artifactories with authentication
  - handled via `.netrc` and mount it in build time via [Buildkit][docker-buildkit-getting-start]
  - no credential exposed in image history or saved in image with [Secret Mount][secret-mount]
  - Added Line

    ```docker
    RUN --mount=type=secret,id=netrc,target=/root/.netrc
    ```

  - After update, updating and installing packages steps

    ```docker
    # securely copy .netrc using BuildKit secrets
    RUN --mount=type=secret,id=netrc,target=/root/.netrc \
        apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        && apt-get clean \
        && rm -rf /var/lib/apt/lists/*
    ```

## Resources

- [Dockerize Python Application][dockerize-python-application]
- [Dockerfile Reference][dockerfile-reference]
- [Heredoc in Dockerfile][heredoc-dockerfile]
- [Custom Dockerfile Syntax][custom-dockerfile-syntax]
- [Deb822-style Format][deb822-style-format]
- [Docker build Concepts][docker-build-concepts]
- [Python cmdline & environments][command-line-and-environment]
- [PIP Configuration][pip-configuration]

[heredoc-dockerfile]: https://www.docker.com/blog/introduction-to-heredocs-in-dockerfiles/
[dockerfile-reference]: https://docs.docker.com/reference/dockerfile/
[custom-dockerfile-syntax]: https://docs.docker.com/build/buildkit/frontend/
[dockerize-python-application]: https://labs.iximiuz.com/challenges/dockerize-python-application
[deb822-style-format]: https://repolib.readthedocs.io/en/latest/deb822-format.html#deb822-style-format
[docker-build-concepts]: https://docs.docker.com/build/concepts/overview/
[docker-buildkit-getting-start]: https://docs.docker.com/build/buildkit/#getting-started
[python-unbuffered]: https://docs.python.org/3/using/cmdline.html#envvar-PYTHONUNBUFFERED
[python-dont-write-bytecode]: https://docs.python.org/3/using/cmdline.html#envvar-PYTHONDONTWRITEBYTECODE
[command-line-and-environment]: https://docs.python.org/3/using/cmdline.html#command-line-and-environment
[pip_install]: https://pip.pypa.io/en/latest/cli/pip_install/
[pip-configuration]: https://pip.pypa.io/en/stable/topics/configuration/
[disabling-caching]: https://pip.pypa.io/en/latest/topics/caching/#disabling-caching
[cache-mount]: https://docs.docker.com/build/cache/optimize/#use-cache-mounts
[copy-virtual-env]: https://pythonspeed.com/articles/multi-stage-docker-python/
[pod-security-context]: https://kubernetes.io/docs/tasks/configure-pod-container/security-context/#supplementalgroupspolicy
[dockerfile-no-new-syntax]: https://github.com/cod3mas0n/Dockerfiles/blob/main/Offline-Dockerfiles/Dockerfile
[secret-mount]: https://docs.docker.com/build/building/secrets/#secret-mounts
[python-env-var-in-dockerfile]: https://dev.to/alimehr75/python-env-variables-in-dockerfile-5920
