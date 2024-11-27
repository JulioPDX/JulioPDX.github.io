---
title: "Codespaces for Network Engineers and Educators"
date: 2024-11-25T15:08:58-07:00
draft: true
toc: true
mermaid: true
images:
- /blog/images/greg-rakozy.jpg
tags:
  - OSPF
  - GitHub
  - Arista
  - Marp
  - Containerlab
---

## Introduction

Can you believe it's been almost a year since my last blog? It's incredible how fast time flies. Thank you all for continuing to read the blog and share it with engineers in the community. I can't thank you enough for all the support and encouragement I've received on this journey.

This blog is going to take a bit of a turn. I usually write in the form of a tool I have used or been presented with and think, "This is incredible; more people should know about it." My other perspective is, "Can I explain this thing in a somewhat more consumable way." This stems from the idea that to really challenge your understanding, teach.

Teaching leads us to the idea behind this blog. I was recently at AutoCon 2 by [Network Automation Forum](https://networkautomation.forum/) (great event) and I met an interesting individual from New Mexico. I won't say their name publicly, but let's call them Bob for this blog. Bob is an instructor at a college and teaches young future engineers all things networking. Their students go through Introductory courses (think CCNA level). However, Bob has an interesting challenge: they want their students to continue learning advanced networking concepts. Bob has a tight budget (if any) and would like to avoid managing a fleet of resources to manage students' lab environments.

You all know Containerlab was at the forefront of my mind. Containerlab and a concept a colleague of mine has been working on to build reproducible labs with GitHub Codespaces got my brain spinning on a possible solution for Bob and their students.

## The Idea Behind Codespaces

Some of you may be wondering, "What a Codespace?" Think of a Codespace as an environment running on GitHub's infrastructure. Another way to look at it is like a portable development environment that can be accessed by a web browser. In its most basic form, all you need to access a Codespace is a GitHub account (free), computer, and internet access. Please see the [official documentation](https://github.com/features/codespaces) for more information. The image below was inspired by the official documentation. In this example, our Codespace would be the box highlighted in red.

![Codespaces infrastructure image](/blog/images/csfne/d2-codespace.svg)

## Codespaces for Network Engineers and Educators

I started pondering what a Network Engineer and Educator would need in a Codespace environment. For starters we would definitely need network nodes in the environment. Reading about concepts is great but seeing them come to life is even more powerful (in my opinion). We should also make the environment easy to use for anyone at any stage of their journey. Because styling is life, it must be pretty. Lastly, we need to be able to present the information to students in something other than a README file (even though I love a good README).

## Building a Codespace

Below is an example of our Codespaces project. I'll go over most files to get you an idea on the process to build an environment like this.

```shell
> tree
.
├── .devcontainer
│   ├── devcontainer.json
│   └── Dockerfile
│   └── postCreate.sh
├── ne101
│   └── 01-routing-ospf
│       └── lesson.md
├── startup
│   ├── r1.cfg
│   └── r2.cfg
└── topo.yml
```

### .devcontainer/devcontainer.json

The .devcontainer directory will host most of our workflow for this environment. For starters we have a `devcontainer.json` file. If your project didn't require a lot of extra tooling (maybe a Python project), the file could look something like the following.

```json
// For format details, see https://aka.ms/devcontainer.json. For config options, see the
{
  "name": "Python 3",
  "image": "mcr.microsoft.com/devcontainers/python:0-3.11-bullseye",
}
```

The example above is using a pre-built container image hosted by Microsoft. In this case, the project is leveraging Python 3.11. The example below provides a more in-depth and realistic view of a `.devcontainer.json` file.

```json
{
    "build": { "dockerfile": "Dockerfile" },
    "hostRequirements": {
        "cpus": 4,
        "memory": "16gb",
        "storage": "32gb"
    },
    "containerEnv": {
        "ARISTA_TOKEN": "${localEnv:ARTOKEN}",
        "CEOS_LAB_VERSION": "4.32.2.1F"
    },
    "features": {
        "ghcr.io/devcontainers/features/github-cli:1": {},
        "ghcr.io/devcontainers/features/docker-in-docker:2": {}
    },
    "secrets": {
        "ARTOKEN": {
            "description": "token to auto-download EOS images from arista.com."
        }
    },
    "customizations": {
        "vscode": {
            "extensions": [
                "catppuccin.catppuccin-vsc",
                "catppuccin.catppuccin-vsc-icons",
                "fabiospampinato.vscode-terminals",
                "marp-team.marp-vscode"
            ]
        }
    },
    "postCreateCommand": "postCreate.sh",
    "postStartCommand": "sudo clab deploy -t topo.yml --reconfigure"
}
```

### build and a Dockerfile

The first portion of the JSON file calls out a build step. We are not using a pre-built container image to load our environment. We will load a Dockerfile local to our project to spin up the environment and install the required tooling. We could in theory build our container image outside of this workflow and call it with the `image` key you saw earlier. Below is a quick view of our `Dockerfile`.

```Docker
FROM mcr.microsoft.com/devcontainers/base:ubuntu

ARG CEOS_LAB_VERSION_ARG

ENV CEOS_LAB_VERSION=${CEOS_LAB_VERSION_ARG}

# install some basic tools inside the container
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    sshpass \
    iputils-ping \
    htop \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/* \
    && rm -Rf /usr/share/doc && rm -Rf /usr/share/man \
    && apt-get clean

# copy postCreate script
COPY ./postCreate.sh /bin/postCreate.sh
RUN chmod +x /bin/postCreate.sh

USER vscode

# install the latest containerlab and eos-downloader
RUN bash -c "$(curl -sL https://get.containerlab.dev)" \
    && pip3 install --user "eos-downloader>=0.10.1"

```

In summary, this file is performing the following:

- Start FROM the base Ubuntu image hosted by Microsoft
- We use the ARG key to pass data into the build environment (in our case this comes from the `.devcontainer.json` file)
- We then set an ENV variable that will be available within the container (notice that we pass along the ARG we set earlier)
- We then RUN a set of commands to install basic tooling like ping and Python3.
- We then COPY a file called `postCreate.sh` (more on this later)
- Towards the end we have one more RUN command, this ensures we have [Containerlab](https://containerlab.dev/) and [eos-downloader](https://github.com/titom73/eos-downloader) installed in our environment

### hostRequirements

When starting a Codespace, you have the option to size it to one of the option given to us by GitHub. The `hostRequirements` key allows us to set minimum values that are required to run this lab. If you don't have the option to use a larger Codespace, this may require contacting them to give you access to larger images (still free but uses more free credits per month).

```json
    "hostRequirements": {
        "cpus": 4,
        "memory": "16gb",
        "storage": "32gb"
    },
```

### containerEnv and secrets

I'll combine these two sections. `contanierEnv` sets environment variables that our container will have access to. One is used as a token to download our Arista cEOS images. If you would like to leverage the Codespace in this example, please create an account with a business email on [arista.com](https://www.arista.com/en/) (free). The other variable will be used in a future workflow to specify which image we would like to download.

```json
    "containerEnv": {
        "ARISTA_TOKEN": "${localEnv:ARTOKEN}",
        "CEOS_LAB_VERSION": "4.32.2.1F"
    },
    "secrets": {
        "ARTOKEN": {
            "description": "token to auto-download EOS images from arista.com."
        }
    },
```

The `ARTOKEN` secret will be available to our container or future `.devcontainer` build steps. The screenshot below shows an example before setting any value for `ARTOKEN`.

![ARTOKEN set](/blog/images/csfne/csfne-artoken.png)

### features

`features` is another incredible feature of Codespaces and devcontainers. We can essentially tell our environment what extra features we would like installed. Think of these as premade scripts that handle the install for us. We add the GitHub CLI feature to simplify our future workflows. Maybe the students aren't git wizards and need a more simple interface to interact with git. We also install the `docker-in-docker` feature to allow us to work with Containerlab. Features are incredibly powerful, if you would like to see a list of features, please see the [official documentation](https://containers.dev/features).

```json
    "features": {
        "ghcr.io/devcontainers/features/github-cli:1": {},
        "ghcr.io/devcontainers/features/docker-in-docker:2": {}
    },
```

### customizations.vscode.extensions

These extensions allow us to extend the functionality or look of VS Code. In this case I am adding my current favorite theme extensions: `catppuccin`. If you were say a University or Company, you could create a custom extension with your own styling. I also load a terminals extension that will give our students the ability to quickly open terminals to the different nodes in the environment. I also install the Marp extension but more on that later.

```json
    "customizations": {
        "vscode": {
            "extensions": [
                "catppuccin.catppuccin-vsc",
                "catppuccin.catppuccin-vsc-icons",
                "fabiospampinato.vscode-terminals",
                "marp-team.marp-vscode"
            ]
        }
    },
```

### .vscode/settings.json

A small break to take a look at our `settings.json` file. Again this can be very deep and complicated but for this example we kept it as simple as possible.

```json
{
    "workbench.colorTheme": "Catppuccin Frappé",
    "workbench.iconTheme": "catppuccin-frappe",
    "markdown.marp.themes": [
        "https://raw.githubusercontent.com/mastern2k3/marpit-nord-theme/master/build/nord-theme.css"
    ]
}
```

Here we set the theme of our VS Code instance and our icon theme. We also set our Marp theme (again explained in a moment). The final version has a lof more Marp themes but that was more for exploration and testing.

### .vscode/terminals.json

Another small break, the code block below shows the configuration definition of our Terminals extension. We set the extension to not autorun and then define the terminals as a list of dictionaries. We can then provide some basic information like the name, icon, color, and command to run on the new terminal. The image below shows an example when running `Terminals: Run` from the Codespaces command palette.

```json
{
    "autorun": false,
    "autokill": true,
    "terminals": [
      {
        "name": "r1",
        "description": "r1 view",
        "icon": "code",
        "color": "terminal.ansiBlue",
        "command": "ssh student@r1",
        "recycle": false,
        "open": true,
        "focus": true
      },
      {
        "name": "r2",
        "description": "r2 view",
        "icon": "code",
        "color": "terminal.ansiRed",
        "command": "ssh student@r2",
        "recycle": false,
        "open": true,
        "focus": false
      }
    ]
}
```

The gif below shows an example when running `'cmd/ctrl+alt+t'` from VS Code to connect to any pre-defined terminal or running `Terminals:run` to connect to all defined terminals.

![Terminals extension gif](/blog/images/csfne/csfne-terminals.gif)

### postCreateCommand

The following portion will run after the container has finished building. Nothing too fancy, we are essentially using eos-downloader to get a specific version of cEOS and having it run the Docker import step for us. We then immediately tag the image with `ceos:latest`.

```json
    "postCreateCommand": "postCreate.sh",
```

```bash
#!/usr/bin/env bash

set +e

ardl get eos --image-type cEOS --version ${CEOS_LAB_VERSION} --import-docker
docker tag arista/ceos:${CEOS_LAB_VERSION} ceos:latest

```

### postStartCommand

We made it to the final portion of the Codespace build, deploying the lab. No surprise here as we are running the Containerlab deploy command to ensure our cEOS nodes are running. We run it with the `--reconfigure` flag to ensure we have no issues when the Codespace shuts down and turns back on.

The only things of note here is that we created startup configurations to set management network stuff in a separate VRF and a student user that did not require passwords to connect. Those can be seen in the `startup` directory.

```json
    "postStartCommand": "sudo clab deploy -t topo.yml --reconfigure"
```

You can see some of the Containerlab output in the Gif above.

## Slides with Marp

We are almost there. We now have an environment to learn about networking concepts or maybe even network automation. In my mind an instructor needs a tool to present this information. Again, I love a good README but giving that to a student and telling them "good luck" is going to have a wide degree of results.

In comes Marp. Marp tags itself as a Markdown Presentation Ecosystem. That sounds fancy but at its core it allows us to write presentations in Markdown format. I wrote about a distant relative [slidev](https://juliopdx.com/2022/11/01/slidev/) in an earlier blog. For this example I felt slidev was a bit heavy and all that I needed to get Marp going was a VS Code extension.

Below is an example of the slides when rendered within the Codespace. You can find this presentation in `ne101/01-routing-ospf/lesson.md` file.

## An Example Lesson

## Outro and Links
