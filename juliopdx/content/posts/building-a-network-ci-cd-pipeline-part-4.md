---
title: "Building a Network CI/CD Pipeline Part 4"
date: 2021-10-31T11:26:48-08:00
draft: false
toc: true
images:
  - /blog/images/masaaki-komori.jpg
tags:
  - CI/CD
  - Batfish
  - NAPALM
  - Nornir
  - Network Automation
  - Python
aliases:
  - /2021/10/31/building-a-network-ci-cd-pipeline-part-4/
---

![Fish](/blog/images/masaaki-komori.jpg)

## Introduction

Hello everyone and thank you for checking out part four in this series. I went on vacation for a bit, but I’m glad to be back on the keys. In this post I will break down all of the steps performed before a change hits our network devices. This is important because we have the opportunity to stop incorrect or invalid configurations from ever hitting our network devices. A few steps that we will cover are: Black for code formatting, Batfish to validate configuration updates, and NAPALM dry run for testing the legitimacy of the configuration files.

![High Level Design](/blog/images/ci_cd_blog.png)

## Black Code Style

When adding steps to this pipeline I have concentrated on easy wins and easy tools to add to the chain. One of the slickest of them being Black. Black is a code formatter based on the Python [PEP 8](https://www.python.org/dev/peps/pep-0008/) standard. You can see the steps in the pipeline file below.

`.drone.yml`

```yaml
steps:
- name: Black Code Format Check
  image: juliopdx/netauto
  commands:
  - black . --check
```

Black will traverse all Python files in our directory and fail if any changes are required. The “–check” option will trigger the failure in our pipeline. Below is an example of what a failure would look like. I added a new file to our repository with a long list variable.

`my_list.py`

```python
my_list = ["something1", "something2", "something3", "something4", "something5", "something6"]
```

I will commit and push this up to our dev branch.

![Pipe Fail](/blog/images/pipe_fail.png)

The Black format check has failed and listed what files would need to be updated. This is very useful to maintain some set standards for yourself or the team on the code style you choose to follow. Below is an example of a success, in this case I have updated that new Python file. Another option to add in this step is something like Pylint. At this time I have not added this to the pipeline but I encourage the reader to check it out. Pylint will go as far as giving the code a score and provide improvement options.

`my_list.py updated`

```python
my_list = [
    "something1",
    "something2",
    "something3",
    "something4",
    "something5",
    "something6",
]
```

![Black Fixed](/blog/images/black_fixed.png)

## Validating Changes With Batfish

Batfish dubs itself as “an open source network configuration analysis tool”. Batfish can validate pre-deployment checks by modeling the network from a source of configurations. This could vary from mismatched BGP neighbor settings, OSPF mismatches, and many more. I’ll walk through installing the Batfish service and run through some examples of what it can find!

Lets get Batfish up and running. Thankfully this can be done by running the latest docker container. This is building on what we have already created from previous articles!

`Batfish Docker Run`

```shell
docker run --name batfish \
    -v batfish-data:/data  \
    -p 9997:9997 -p 9996:9996 \
    -d batfish/batfish
```

Great, now that the service is running we need to create a configuration snapshot. Think of this as our production network or the changes we would like to push to our production network. This could be one site or the whole enterprise. Be mindful, you may need more horsepower if you are testing against a large network. In our case we have a very simple four node topology running OSPF and BGP between R1 and R4. To get the configuration snapshot up and running I created a simple backup script to store the files in the correct location.

`Directory Structure`

```shell
(venv) juliopdx@juliopdx-pop:~/git/ci_cd_dev$ tree snapshots/
snapshots/
└── configs
    ├── pdx-rtr-eos-01.txt
    ├── pdx-rtr-eos-02.txt
    ├── pdx-rtr-eos-03.txt
    └── pdx-rtr-eos-04.txt
```

## Batfish Assertion Helpers

While exploring Batfish, I ran into their [assertion helpers](https://batfish.readthedocs.io/en/latest/asserts.html) page. There is some gold in that page and I highly recommend you check it out. I used these heavily in this test deployment to see what errors Batfish would find in the configuration. Pretty much all of these require the same bit of information (a snapshot is required). I’ll walk through the script and one of the assert functions, the rest follow the same path.

Below are some of the standard pybatfish imports followed by a decent amount of the assert helpers. Importing Rich because it’s awesome and I like colors.

`Imports`

```python
from pybatfish.client.commands import *
from pybatfish.question import load_questions
from pybatfish.client.asserts import (
    assert_no_duplicate_router_ids,
    assert_no_incompatible_bgp_sessions,
    assert_no_incompatible_ospf_sessions,
    assert_no_unestablished_bgp_sessions,
    assert_no_undefined_references,
)
from rich import print as rprint
```

Below is the function used to test if we have duplicate router IDs. It requires one argument of snap or snapshot.

```python
def test_duplicate_rtr_ids(snap):
    """Testing for duplicate router IDs"""
    assert_no_duplicate_router_ids(
        snapshot=snap,
        protocols={"ospf", "bgp"},
    )
```

This function will look at BGP and OSPF within our snapshot to see if duplicate router IDs are set. I omitted a few Rich prints because they are not pertinent to the functionality of the function. Below if the main function that calls all of our assert helpers. A lot of this is created from samples in the Batfish documentation. For example, we name our snapshot and point it to the correct directory. We also point the script to the correct IP that is running the service.

`Main Function`

```python
def main():
    """init all the things"""
    NETWORK_NAME = "PDX_NET"
    SNAPSHOT_NAME = "snapshot00"
    SNAPSHOT_DIR = "./snapshots"
    bf_session.host = "192.168.10.184"
    bf_set_network(NETWORK_NAME)
    init_snap = bf_init_snapshot(SNAPSHOT_DIR, name=SNAPSHOT_NAME, overwrite=True)
    load_questions()
    test_duplicate_rtr_ids(init_snap)
    test_bgp_compatibility(init_snap)
    test_ospf_compatibility(init_snap)
    test_bgp_unestablished(init_snap)
    test_undefined_references(init_snap)


if __name__ == "__main__":
    main()
```

## Test for Duplicate Router IDs

All of our routers have RIDs set as 10.0.0.x, x being the router number. Lets change the router ID under OSPF for R1 to be 10.0.0.2, this is done under the snapshot directory configuration file for R1. For an infrastructure as code build these would be created from YAML files or some source of truth and maybe some jinja sprinkled in there. This change would be a duplicate of R2 and we should see the pipeline fail.

`R1 OSPF Configuration`

```text
router ospf 1
   router-id 10.0.0.2
   passive-interface Ethernet2
   passive-interface Loopback1
   max-lsa 12000
```

We will commit and push this to our dev branch, the pipeline output is below. In the bottom right you can barely see the duplicate router IDs error and towards the bottom of the output it will list the devices that are in error. In this case both R1 and R2 have router IDs set to “10.0.0.2”.

![RID Error](/blog/images/rid_error.png)

## Test for Incompatible BGP Sessions

Lets add another error. In this case R1 and R4 are running multihop EBGP. R1 has an autonomous system (AS) of 65001 and R4 has an AS of 65004. Lets just say someone misconfigured BGP and set R1 to have a neighbor of 65003 vs 65004. As we can see R1 is set to have a neighbor with R4 using AS 65003 but R4 is configured with AS 65004. You may have noticed towards the top of the pipeline that no duplicate router IDs were found in this run.

`R1 Bad BGP`

```text
router bgp 65001
   router-id 10.0.0.1
   neighbor 10.0.0.4 remote-as 65003
   neighbor 10.0.0.4 update-source Loopback1
   neighbor 10.0.0.4 ebgp-multihop 3
```

![BGP Error](/blog/images/bgp_error.png)

## Test for OSPF Network Mismatches

Okay this is the last example we will use but I think you all get the idea. All OSPF neighbors have interfaces configured with point to point interfaces. Lets remove the R1 to R2 point to point interface configuration in our snapshot. This should result in a network type mismatch error.

```text
interface Ethernet1
   no switchport
   ip address 10.0.12.1/24
 - ip ospf network point-to-point
   ip ospf area 0.0.0.0
```

![OSPF Error](/blog/images/ospf_error.png)

At this point we have prevented a few configuration errors from entering the network. Please note, this is only scratching the surface on what can be done with Batfish. Please check out their documentation and code examples on more ideas to test your network. Links at the end of this post.

## Validating Configuration With NAPALM Dry Run

Batfish is great and it can catch a lot of errors or misconfigurations. What if there is something that can’t be caught by Batfish as a configuration error. How would we validate what we are sending is even a valid configuration in general? In this case I leveraged Nornir with NAPALM. NAPALM has a “napalm_configure” task that will attempt to send a complete configuration file to a network device. If it is not valid, it should report an error. Remember that we are in the precheck stage and don’t want any actual changes to hit our network devices. Below is a snippet on what I came up with to keep the code small but also functional between precheck deployments and actual deployments.

`build.py`

```python
import argparse
from nornir import InitNornir
from nornir_napalm.plugins.tasks import napalm_configure
from nornir_utils.plugins.functions import print_result
from tools import nornir_set_creds


parser = argparse.ArgumentParser()
parser.add_argument(
    "--dry_run", dest="dry", action="store_true", help="Will not run on devices"
)
parser.add_argument(
    "--no_dry_run", dest="dry", action="store_false", help="Will run on devices"
)
parser.set_defaults(dry=True)
args = parser.parse_args()


def deploy_network(task):
    """Configures network with NAPALM"""
    task1_result = task.run(
        name=f"Configuring {task.host.name}!",
        task=napalm_configure,
        filename=f"./snapshots/configs/{task.host.name}.txt",
        dry_run=args.dry,
        replace=True,
    )


def main():
    """Used to run all the things"""
    norn = InitNornir(config_file="configs/config.yaml", core={"raise_on_error": True})
    nornir_set_creds(norn)
    result = norn.run(task=deploy_network)
    print_result(result)


if __name__ == "__main__":
    main()
```

The top portions of the script import anything required to interact with Nornir and NAPALM. We then use “argparse” to create an argument with the script that will set a variable to True or False. This can then be used during the “napalm_configure” task to either run in dry mode or actually implement changes. Below is how the precheck looks like in the “.drone.yml” file.

`Configuration Precheck`

```yaml
- name: Precheck Configuration Diff
  image: juliopdx/netauto
  environment:
    MY_SECRET:
      from_secret: MY_SECRET
  commands:
  - python build.py --dry_run
```

## NAPALM Dry Run With Error

Below is an example of adding something to the configuration that will definitely not work and how it looks like in the pipeline.

`Error Configuration`

```text
router bgp 65001
   router-id 10.0.0.1
   neighbor 10.0.0.4 remote-as 65004
   neighbor 10.0.0.4 update-source Loopback1
   neighbor 10.0.0.4 ebgp-multihop 3
!
something super fake
   welcome to the world of tomorrow
   its over 9000
   router-id infinity
!
router ospf 1
   router-id 10.0.0.1
   passive-interface Ethernet2
   passive-interface Loopback1
   max-lsa 12000
!
```

![Invalid Config](/blog/images/invalid_config.png)

## NAPALM Dry Run With Valid Configuration

Below is a standard change, adding a simple description to an interface. Included pipeline output as well.

`Valid Configuration`

```text
!
interface Ethernet1
   description Welcome to the world of tomorrow!
   no switchport
   ip address 10.0.12.1/24
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
```

![Valid Configuration](/blog/images/valid_config.png)

## Outro and Links

Whats included in this post is just a subset of options that are available in prechecks. We could add ACL, input validation, Pylint, routing tests, and more. I hope what’s included here sparks some ideas or gives you something to add to your CI/CD workflow. I think in the next post we will go over the tool used to actually deploy configurations once they have passed our prechecks. In our case Nornir, but this could be Ansible, Scrapli, or some other solution.

- [Featured Image by Masaaki Komori](https://unsplash.com/photos/vXid97obEy8)
- [Black Code Formatting](https://black.readthedocs.io/en/stable/)
- [Batfish and pybatfish](https://pybatfish.readthedocs.io/en/latest/)
- [Nornir NAPALM](https://github.com/nornir-automation/nornir_napalm)
- [Building a Network CI/CD Pipeline Part 1](https://juliopdx.com/2021/10/20/building-a-network-ci/cd-pipeline-part-1/)
- [Building a Network CI/CD Pipeline Part 2](https://juliopdx.com/2021/10/20/building-a-network-ci/cd-pipeline-part-2/)
- [Building a Network Ci/CD Pipeline Part 3](https://juliopdx.com/2021/10/20/building-a-network-ci/cd-pipeline-part-3/)
- [Building a Network CI/CD Pipeline Part 5](https://juliopdx.com/2021/11/08/building-a-network-ci/cd-pipeline-part-5/)
- [Building a Network CI/CD Pipeline Part 6](https://juliopdx.com/2021/11/12/building-a-network-ci/cd-pipeline-part-6/)
