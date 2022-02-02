---
title: Build an Ansible Execution Environment
date: 2022-02-01
categories: [Automation]
tags: [ansible, devops, awx]     # TAG names should always be lowercase
---

## Overview

Recently, Ansible has released a tool to build docker image with all your requirements. Main goal is to support execution of projects in [Ansible AWX](https://docs.ansible.com/automation-controller/latest/html/userguide/execution_environments.html) and to replace the need to configure Python virtual-environment.

In addition, it is very convenient to leverage this function to build test environment for your ansible development or to ship a feature for testing.

In this article, we will see how to build such images and how to use these images.

## Pre-requisites

Before we start, you need to have a docker or a podman running on your machine.

- [Docker desktop](https://docs.docker.com/get-docker/)
- [Podman](https://podman.io/getting-started/installation)

Also python in version `3.x` is highly recommended.

## Configure your Ansible Execution Environment

Instead of using `Dockerfile` syntax, ansible EE is based on a very simple YAML schema to define following elements:

- Base image from [ansible](https://quay.io/repository/ansible/ansible-runner?tag=latest&tab=tags) to use: It is where you will specify your ansible version
- List of python and ansible requirements.
- A list of [`Dockerfile` commands](https://docs.docker.com/engine/reference/builder/) to run __before__ and/or __after__ the main part of the build

Snippet below is an example of such file:

```yaml
# execution-environment.yml
---
version: 1

build_arg_defaults:
  EE_BASE_IMAGE: 'quay.io/ansible/ansible-runner:stable-2.12-devel'

dependencies:
  galaxy: requirements.yml
  python: requirements.txt

additional_build_steps:
  prepend: |
    RUN pip install --upgrade pip setuptools
    RUN yum install -y make wget curl less git vim sshpass
  # append:
  #   - RUN ls -la /etc
```

And in the same directory, you also have to put your requirements files:

```bash
ls -l
execution-environment.yml
requirements.txt
requirements.yml
```

We won't go through the python requirements file as it is a standard one, but we can take few seconds to look at the Ansible requirement file:

```yaml
---
collections:
  - name: titom73.avd_tools
    version: 0.2.0
  - name: ansible.netcommon
  - ansible.posix
  - community.general
```

As you can see, we can optionally defined version for collection. And we can also use git as source of installation as shown in example below:

```yaml
collections:
  - name: arista.avd
    source: git+https://github.com/titom73/ansible-inetsix.git#/ansible_collections/titom73/avd_tools/,devel
```

As you can see, URL is specific and contains the following blocks:

- Git repo: `git+https://github.com/aristanetworks/ansible-avd.git`
- Path for the collection: `#/ansible_collections/arista/avd/`
- Git Branch to consider: `,devel`

> More information available on [Ansible website](https://docs.ansible.com/ansible/latest/user_guide/collections_using.html#installing-a-collection-from-source-files)

## Build Ansible Execution Environment

To build this image, you need to install a tool from ansible named `ansible-builder` available via `pip`

```bash
$ pip3 install ansible-builder
```

This tool is in charge to read your configuration file, generate a Containerfile or a Dockerfile and execute the build using either podman (default) or docker with tag you will provide via the CLI.

### Build using podman

```bash
# Build image
$ ansible-builder build -f execution-environment.yml \
    --tag titom73/ansible-ee:2.12
Running command:
  podman build -f context/Containerfile -t titom73/ansible-ee:2.12 context
Complete! The build context can be found at: \
    /Users/titom73/Projects/avd-stack/avd-lab-validation/context

# Check result
podman images
REPOSITORY                       TAG                IMAGE ID      CREATED             SIZE
localhost/titom73/ansible-ee     2.12               9fd6fc5264f4  54 seconds ago      1.15 GB
quay.io/ansible/ansible-runner   stable-2.12-devel  5e00350a4bb5  20 hours ago        881 MB

# Check your requirements are installed
podman container run -it --rm \
    localhost/titom73/ansible-ee:2.12 ansible-galaxy collection list

# /usr/share/ansible/collections/ansible_collections
Collection        Version
----------------- -------
ansible.netcommon 2.5.0
ansible.posix     1.3.0
ansible.utils     2.4.3
community.general 4.4.0
titom73.avd_tools 0.2.0
```

### Build with Docker engine

Process is exactly the same, except you have to configure `--container-runtime` option to use docker

```bash
# Build image
$ ansible-builder build -f execution-environment.yml \
    --tag titom73/ansible-ee:2.12 \
    --container-runtime docker
Running command:
  docker build -f context/Dockerfile -t titom73/ansible-ee:2.12 context

# Check result
docker images
REPOSITORY           TAG      IMAGE ID       CREATED              SIZE
titom73/ansible-ee   2.12     fdcdd11ea397   About a minute ago   1.11GB

# Check your requirements are installed
docker run -it --rm \
    localhost/titom73/ansible-ee:2.12 ansible-galaxy collection list

# /usr/share/ansible/collections/ansible_collections
Collection        Version
----------------- -------
ansible.netcommon 2.5.0
ansible.posix     1.3.0
ansible.utils     2.4.3
community.general 4.4.0
titom73.avd_tools 0.2.0
```

## Hack the build.

This section is more about tips & tricks around ansible-builder

### Build container with local collection

If you are developing a feature and need someone to test it, you can build image with your local collection

```yaml
# execution-environment.yml
---
version: 1

build_arg_defaults:
  EE_BASE_IMAGE: 'quay.io/ansible/ansible-runner:latest'

dependencies:
  galaxy: requirements.yml
  python: requirements.txt

additional_build_steps:
  prepend: |
    RUN pip install --upgrade pip setuptools
    RUN yum install -y make wget curl less git vim sshpass
  append:
    - COPY ansible_collections/titom73/avd_tools/ \
        /usr/share/ansible/collections/ansible_collections/titom73/avd_tools
    - RUN ls -la /usr/share/ansible/collections/ansible_collections/titom73
```

> Assuming path to your local collection is ansible_collections/titom73/avd_tools/

### Generic Make

```make
EE_IMAGE ?= titom73/ansible-ee
EE_TAG ?= 2.12

.PHONY: ee-build
ee-build: ## Build Ansible Execution Builder
    ansible-builder build \
        --tag $(EE_IMAGE):$(EE_TAG) \
        --container-runtime docker \
        -f $(EE_FILE) \
        --build-arg EE_BASE_IMAGE=quay.io/ansible/ansible-runner:stable-$(EE_TAG)-devel
```

So easy to use:

```bash
make ee-build EE_TAG=2.11
```

## Additional Resources

- Ansible Builder [documentation](https://ansible-builder.readthedocs.io/en/stable/)
- Network to Code: [Ansible Builder, Runner, and Execution Environments](https://blog.networktocode.com/post/ansible-builder-runner-ee/)
- Ansible Runner [documentation](https://ansible-runner.readthedocs.io/en/stable/intro/)
