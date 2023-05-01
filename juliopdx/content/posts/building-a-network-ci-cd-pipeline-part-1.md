---
title: "Building a Network CI/CD Pipeline Part 1"
date: 2021-10-20T10:26:36-08:00
draft: false
toc: true
images:
  - /blog/images/selim.jpg
tags:
  - CI/CD
  - Network Automation
  - Docker
aliases:
    - /2021/10/20/building-a-network-ci-cd-pipeline-part-1/
---

![Featured Image](/blog/images/selim.jpg)

## Introduction

Hello all and thank you for joining me on another blog post! In this post or series of posts I hope to walk you through my journey on building a network CI/CD pipeline. This pipeline is not perfect and I’m sure there’s much more efficient ways to produce the same outcomes. I did make the choice to leave out a few things as the technologies involved and complexity kept growing. For example, I did not go deep into infrastructure as code with this example but I will link a great series from a fellow network engineer at the bottom of this post.

I tried to make this deployment as simple as possible and a few of the technologies are very new to me. I say this because I feel a lot of us get lost in the sauce when so many new technologies are mentioned or even displayed in one architecture diagram. I hope to minimize some of that restraint and break the pieces down as much as possible.

## What is CI/CD?

Some of you might be wondering what CI/CD is or even why we should care? CI/CD is a software development concept where software is created, tested and eventually released in a continuous manner. The pieces involved for different applications may differ but I hope to show you some of the main concepts.

How does this relate to network engineering? Lets say you didn’t have a CI/CD pipeline and you wanted to make a change on the production network. The steps would be something like the following:

- User requests change to the network
- Change control ticket is created detailing change and test plan
- Login to a test environment if there is one and test the change
- Login to the devices one by one and implement change
- Validate expected outcomes after change on the devices

This process is fairly slow and has a lot of touch points. For example, the testing plan by different engineers may differ or a missed validation could lead to a network outage or an unexpected behavior from the change. Using a CI/CD pipeline we hope to automate a lot of these steps to ensure the change is correct and safe to implement on our network. This isn’t to say that a pipeline will eliminate all errors, far from that. This will allow us to continually improve our testing and the network over time to make it easier to manage and implement changes quickly and safely.

## Pieces to the Puzzle

Below are some of the technologies we will be breaking down over this series of posts:

- GitHub
- Docker
- Drone(pipeline server and runners)
- NGROK
- Batfish
- Nornir/NAPALM
- Suzieq

![High Level Design](/blog/images/ci_cd_blog.png)

## Getting Started

Please note, the local code development can be done on any laptop or PC of choice. The docker pieces can be done on a local machine as well but I chose to stand up a simple Ubuntu 20.04 virtual machine with 4 CPUs and 8G of RAM.

### GitHub

If you are following along or just getting started then I would highly recommend creating a GitHub account. This is a great place to share and collaborate with other creators in a wide variety of spaces. If you are studying a programming language or going down the path of NetDevOps, you will definitely want some version control.

### Docker

I’ll be honest, I’m fairly new to docker and containers as a whole. Imagine a docker container being one of those Lunchables you ate as a kid. Everything you need is inside of that container. No external dependencies are involved. As mentioned before I created an Ubuntu VM for this deployment to run docker and any containers. The first step was getting docker installed. Credit to the team at Digital Ocean for the great write up on installing docker, check it out [here](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04).

```shell
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
apt-cache policy docker-ce
sudo apt install docker-ce
sudo systemctl status docker
```

### Using Docker without Sudo

```shell
sudo usermod -aG docker ${USER}
su - ${USER}
```

### Useful Docker Commands

`View a list of running containers`

```shell
juliopdx@drone:~$ docker ps
CONTAINER ID   IMAGE                         COMMAND                  STATUS         NAMES
ed76314f72f7   drone/drone-runner-docker:1   "/bin/drone-runner-d…"   Up 2 minutes   runner
500785d2e6d4   drone/drone:2                 "/bin/drone-server"      Up 3 minutes   drone
2df0b3ee02fb   batfish/batfish               "java -XX:-UseCompre…"   Up 8 days      batfish
```

`Stop or start containers “docker stop/start <container name>`

```shell
juliopdx@drone:~$ docker stop runner
runner
juliopdx@drone:~$ docker start runner
runner
juliopdx@drone:~$
```

If you want to explore more Docker commands, check out the [official documentation](https://docs.docker.com/engine/reference/commandline/docker/)!

## Outro and Links

I’ll end this one here. Stay tuned for the next post, setting up the Drone server and runners!

- [Infrastructure as code series by NWMichl](https://nwmichl.net/2020/10/28/network-infrastructure-as-code-with-ansible-part-1/)
- [Installing Docker on Ubuntu 20.04 by Digital Ocean](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-ubuntu-20-04)
- [Featured image by @selimarda](https://unsplash.com/photos/Grg6bwZuBMs)
- [Building a Network CI/CD Pipeline Part 2](https://juliopdx.com/2021/10/20/building-a-network-ci/cd-pipeline-part-2/)
- [Building a Network CI/CD Pipeline Part 3](https://juliopdx.com/2021/10/20/building-a-network-ci/cd-pipeline-part-3/)
- [Building a Network Ci/CD Pipeline Part 4](https://juliopdx.com/2021/10/31/building-a-network-ci/cd-pipeline-part-4/)
- [Building a Network CI/CD Pipeline Part 5](https://juliopdx.com/2021/11/08/building-a-network-ci/cd-pipeline-part-5/)
- [Building a Network CI/CD Pipeline Part 6](https://juliopdx.com/2021/11/12/building-a-network-ci/cd-pipeline-part-6/)
