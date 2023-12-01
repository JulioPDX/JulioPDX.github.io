---
title: "Network CI and Open Source"
date: 2023-11-25T19:33:45-08:00
draft: true
toc: true
mermaid: false
images:
  - /blog/images/network-ci/atanas-malamov.jpg
tags:
  - AVD
  - Containerlab
  - GitHub
  - pre-commit
  - ruff
  - Ansible
  - Docker
---

# Introduction

Wow, it's been a long time since my last blog post. I recently attended the first [AutoCon](https://networkautomation.forum/) event focusing on network automation. Long story short, it was terrific. Check it out and be on the lookout for the next one. Shortly after AutoCon, two community members sprung up with sweet blog posts. My colleague and friend Dan Hertzberg are resurrecting the blog. The first post went over leveraging [ygot](https://danielhertzberg.net/posts/ygotpt1/) to create Go structs. My now in-person friend Danny Wade also made a new post that [recapped AutCon](https://devnetdan.com/2023/11/21/naf-autocon0-recap/). Both are excellent reads and good people; check them out.

Shortly after the event, I wanted to run a simple continuous integration pipeline with Containerlab and some Arista cEOS nodes. Long story short, it worked! However, I had to add a twist to the pipeline by leveraging a GitHub self-hosted runner.

If you've read my blogs in the past, you may have run into my six-part blog series on [building a CICD pipeline](https://juliopdx.com/2021/10/20/building-a-network-ci/cd-pipeline-part-1/). Think of this post as a modification or add-on to that previous work. I'll review more tools and provide examples of how they can help you on your network automation journey. I hope you enjoy it.

## The why

If you got this far, then I've either hooked you with the introduction, or you're just a fan of my work. Either way, thank you. The main reason for this post is to look at different open-source tools and see how they can be used in our workflows to improve or automate some of the tasks.

One important note! I'll mention a lot of different tools in this blog, don't feel like you have to use all of them. Feel free to explore and choose what works for you and your environment.

## Requirements

If you want to follow along, you should have a local development machine and a Linux VM or laptop to run the self-hosted runner. In my case, I have it running on a small Linux VM on my Intel NUC (not sponsored). It would be helpful if your development machine has [Docker installed](https://docs.docker.com/engine/install/ubuntu/), but the Linux VM will need it.

### GitHub self-hosted runner

GitHub has robust CI tooling within its platform. We can leverage the tooling to create workflows to automate project tasks. These tasks could be auto-assigning team members to issues, testing code, building artifacts, or validating changes on the network.

These "actions" can run within GitHub's infrastructure to spin up ephemeral environments. This is fantastic, but it could be better when we need this environment to access our containerized network images. The only vendor I am aware of that allows public pulling of their container image is Nokia with their SRLinux platform.

For my setup, I will be using Arista's cEOS images. Instead of hosting the image in a public repository (which I don't think is on the up-and-up), I leveraged the self-hosted runner. Since we manage the self-hosted runner, we can have whatever we want, including container images. More on this in a moment.

GitHub has incredible documentation on [installing a self-hosted runner](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/adding-self-hosted-runners). I won't duplicate that effort, and I'll link you to their excellent docs. One important thing to note is that their documentation walks us through installing the application and running it within the terminal. We need it to run in the background as a service. To do this, make sure you follow [GitHub's docs](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/configuring-the-self-hosted-runner-application-as-a-service) on doing just that. Below is an example output from my VM running the application.

```shell
❯ sudo ./svc.sh status

/etc/systemd/system/actions.runner.JulioPDX-ci-avd-clab.dev.service
● actions.runner.JulioPDX-ci-avd-clab.dev.service - GitHub Actions Runner (JulioPDX-ci-avd-clab.dev)
     Loaded: loaded (/etc/systemd/system/actions.runner.JulioPDX-ci-avd-clab.dev.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2023-11-22 12:28:11 PST; 3 days ago
   Main PID: 1713 (runsvc.sh)
      Tasks: 23 (limit: 4915)
     Memory: 1.6G
        CPU: 44min 55.008s
     CGroup: /system.slice/actions.runner.JulioPDX-ci-avd-clab.dev.service
             ├─1713 /bin/bash /home/juliopdx/actions-runner/runsvc.sh
             ├─1716 ./externals/node16/bin/node ./bin/RunnerService.js
             └─1724 /home/juliopdx/actions-runner/bin/Runner.Listener run --sta…

Hint: Some lines were ellipsized, use -l to show in full.
```

### Containerlab

[Containerlab](https://containerlab.dev/) is by far my favorite platform to run labs. Whether I'm personally learning, testing, or incorporating it into a blog post like this one; the speed and simplicity Containerlab provides is unmatched. If you are following along, feel free to install it with this one-lines.

```shell
bash -c "$(curl -sL https://get.containerlab.dev)"
```

### Arista cEOS images

In one of my earlier [blog posts](https://juliopdx.com/2021/12/10/my-journey-and-experience-with-containerlab/#how-to-add-networking-images-to-containerlab) I talked about how to obtain Arista cEOS images. You basically make a free account on the site, download whatever version you want, and import the image with Docker. Way before I was an Arista employee I appreciated the fact that the EOS images were free to obtain, more vendors should definitely do this. There is also a new tool available called [eos-downloader](https://github.com/titom73/eos-downloader); it is a command line interface to download Arista EOS images. I would encourage you to check it out.

That is all you need on the Linux VM running the GitHub self-hosted runner application. At the end of this blog I'll post a few alternate ways we could improve this workflow, feel free to use that as self learning opportunities.

## The basic workflow

In the most high-level view of our build, we have a local development machine where we can make proposed changes to our test network. The changes in "code" are then pushed up to GitHub where automated workflows will run to lint our code, build the configurations, and test it against a short-lived network.

![Workflow](/blog/images/network-ci/d2.svg)

## But it's so much more

If only the world was that simple. In reality there are a decent amount of tools available to make this all happen. Feel free to fork or clone the [repository](https://github.com/JulioPDX/ci-avd-clab.git) and mess with whatever you'd like to. Below is a view of the folder structure of the repository. Stick with me as I'll do my best to break down each and every piece.

```shell
git clone https://github.com/JulioPDX/ci-avd-clab.git
```

```shell
> tree -I venv
.
├── ansible.cfg
├── anta
│   ├── anta-inv.yml
│   └── anta-test.yml
├── artifacts
│   ├── config_backup
│   ├── documentation
│   ├── intended
│   │   ├── configs
│   │   ├── structured_configs
│   │   └── test_catalogs
│   └── reports
│       └── anta_reports
├── group_vars
│   ├── all.yml
│   ├── ATD_FABRIC.yml
│   ├── ATD_LAB.yml
│   ├── ATD_SERVERS.yml
│   └── ATD_TENANTS_NETWORKS.yml
├── inventory.yml
├── lab.yml
├── LICENSE
├── playbooks
│   ├── build.yml
│   ├── deploy-net.yml
│   ├── validate-net-anta.yml
│   └── validate-net.yml
├── README.md
├── requirements-dev.txt
├── requirements.txt
├── requirements.yml
└── tasks.py
```

### Preparing the local environment

The repository also includes a `.devcontainer` file as well. Very useful if you want to develop within a container environment. I'll walk-through setting up a local Python development environment. This assumes you have Python installed. If you would like to test more in-depth locally, ensure you have Docker, Containerlab, and a cEOS image available.

```shell
cd ci-avd-clab
python3 -m venv venv
source venv/bin/activate
pip3 install -r requirements-dev.txt
```

### GitHub Actions

GitHub Actions is the CI/CD platform within GitHub. We can leverage GitHub Actions to create automated workflows within our repository. These workflows can be as simple linting our codebase for any errors or deploying changes to a test network and notifying us of a pass or fail. Below is the entire look at the GitHub Actions file for this example. The file is located in the `.github/workflows` folder.

```yaml
---
name: Network CI

on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main

jobs:
  run-linters:
    timeout-minutes: 15
    runs-on: ubuntu-latest
    steps:
      - name: checkout
        uses: actions/checkout@v4

      - name: setup python
        uses: actions/setup-python@v4

      - name: run pre-commit
        uses: pre-commit/action@v3.0.0

  dev-build-deploy-test:
    needs: run-linters
    env:
      ANTA_PASSWORD: ${{ secrets.ANTA_PASSWORD }}
      SUDO_PASS: ${{ secrets.SUDO_PASS }}
      CEOS_VERSION: 4.30.2F

    timeout-minutes: 15
    runs-on: avd-ci
    steps:
      - name: checkout
        uses: actions/checkout@v4

      - name: setup Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.9"

      - name: install requirements
        run: pip install invoke && invoke setup-base

      - name: create topology with containerlab
        run: echo $SUDO_PASS | sudo -S containerlab deploy -t lab.yml --reconfigure

      - name: build with AVD
        run: ansible-playbook playbooks/build.yml

      - name: deploy configuration to nodes
        run: |
          ansible-playbook playbooks/deploy-net.yml
          sleep 10

      - name: run ANTA cli tests
        run: |
          anta \
              --username admin \
              --password $ANTA_PASSWORD \
              --inventory anta/anta-inv.yml \
              --log-file artifacts/anta.log \
              nrfu --catalog anta/anta-test.yml text

      - name: validate configuration on nodes
        run: ansible-playbook playbooks/validate-net.yml

      - name: validate configuration on nodes w/anta
        run: ansible-playbook playbooks/validate-net-anta.yml

      - name: destroy lab
        if: always()
        run: echo $SUDO_PASS | sudo -S containerlab destroy -t lab.yml -c

      - name: cleanup self hosted runner
        if: always()
        uses: TooMuch4U/actions-clean@v2.1
```

I'll break this down piece by piece and show you most of the tools from my local development machine along the way.

```yaml
---
name: Network CI

on:
  pull_request:
    branches:
      - main
  push:
    branches:
      - main
...
```

At the root of the GitHub actions file we define a `name` for the workflow. This doesn't have any impact on the workflow itself but it's always helpful to pick a name with meaning. The `on` key is where things start to get interesting. Here we can specify on what events and on what branches we would like this workflow to run on.

At this moment we have it running on a pull request event or a push event, both heading towards the main branch. This definition is not something to use in a production environment. More than likely you would have a workflow file for your feature/fix branches and another for the main branch or the production network.

### Jobs, Linters, and pre-commit

```yaml
...
jobs:
  run-linters:
    timeout-minutes: 15
    runs-on: ubuntu-latest
    steps:
      - name: checkout
        uses: actions/checkout@v4

      - name: setup python
        uses: actions/setup-python@v4

      - name: run pre-commit
        uses: pre-commit/action@v3.0.0
...
```

### The self-hosted runner does a thing

### Containerlab part two

### The AVD framework

### ANTA for network validation

## Outro and links

- [Featured Image Atanas Malamov](https://unsplash.com/photos/lake-surrounded-by-pine-trees-near-snow-covered-mountain-tpmAv6c33dE)
