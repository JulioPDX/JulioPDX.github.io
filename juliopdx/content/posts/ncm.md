---
title: "NATS and Configuration Minions"
date: 2024-12-02T14:53:36-08:00
draft: false
toc: true
mermaid: true
images:
- /blog/images/ncm/jiayuan-lian.jpg
tags:
  - NATS
  - Arista
  - Containerlab
  - AVD
  - NAPALM
---

## Introduction

Two blogs in one year. Who do I think I am? In all seriousness, thank you for all the feedback on the previous blog. As always, it really means a lot. This one might go a bit sideways, but building something like this has always been on my mind, and I got another spark to get it done while attending AutoCon2.

I caught the first half of a talk by Mircea Ulinic from Digitial Ocean. In that talk, Mircea went over how the company leverages automation at an incredible scale. They eventually covered a bit on [Salt](https://github.com/saltstack/salt) and how they leverage proxy minions to effectively have a one-to-one mapping to a networking node. Imagine one proxy being deployed for each device; the proxy would handle the job when a task needs to be performed on said device.

I wanted to try making something similar, but with the huge caveat I would only tackle one portion—configuration management. I know everyone talks about configuration management, but stick with me and let me have a little fun in my small world. This is a breakdown of how I got a minimal viable product (MVP) running.

## What is NATS

I'm not going to bore anyone with a drawn-out definition, but at its core, [NATS](https://nats.io/) is a messaging system that ensures your message will arrive at most once at a subscribed destination. NATS does this in the way of subjects. We might publish a message to `say.hello` and someone else will be subscribed to listen to messages on `say.hello`. The subscriber would then get the message and whatever content is included in the message. The NATS server handles the passing of messages. There you go, NATS in a nutshell or paragraph. Side note, the [NATS documentation](https://docs.nats.io/) is excellent. Please go check it out.

![Example of a NATS workflow](/blog/images/ncm/nats.svg)

## The Workflow

That sounds cool, but again, this blog is focused mainly on Network Engineering, so why do we care? For this example, we will leverage NATS as our messaging system to communicate with our minions or runners or whatever other name you'd like to give them. We'll do this through subjects like you saw earlier, but our subjects will be named after nodes and the action we would like performed.

![Example of a NATS workflow with network devices](/blog/images/ncm/goal.svg)

## The Topology

The topology below looks a bit wild at first glance, but honestly, it's not as wild as some network diagrams I've seen in the past 😅. I'll go over each portion piece by piece.

![The overall topology](/blog/images/ncm/topo.svg)

### The Builder and the Watch Service

Everything in this lab is running on my 2019 Lenovo X1 Carbon laptop. You should be able to run this as well 😅. I think phones are more powerful at this point. The builder (my computer) is running [AVD](https://avd.arista.com/stable/index.html); this isn't required as we're only using AVD to generate our final device configurations. You can use whatever methodology or framework you prefer. AVD will generate device configurations in the `deployment/intended/configs` directory. Each file will be called something like `leaf1.cfg`.

![The builder topology](/blog/images/ncm/builder.svg)

The watch service is fairly short-winded for this example. We leverage the [watchfiles](https://github.com/samuelcolvin/watchfiles) Python package and point it to a directory to look for file changes. Once we see a change, we massage some data to get our device name and read the contents. When this is done, we leverage the [NATS](https://github.com/nats-io/nats.py) Python package to publish a message to the specific subject.

```python
import asyncio
import os

import aiofiles
import nats
from watchfiles import awatch


async def watch_directory(nc, path):
    async for changes in awatch(path):
        for _, file_path in changes:
            async with aiofiles.open(file_path, mode="r") as f:
                contents = await f.read()
                device_name = os.path.splitext(os.path.basename(file_path))
                await nc.publish(f"dc1.{device_name[0]}.configure", contents.encode())


async def main():
    nc = await nats.connect("nats://localhost:4222")
    await watch_directory(nc, "deployment/intended/configs")
    await asyncio.Event().wait()


if __name__ == "__main__":
    asyncio.run(main())

```

![The Watch service](/blog/images/ncm/watch.svg)

### The NATS Server

The NATS Server will handle all the communication between the publishers and subscribers. There's a lot we could do with the configuration, but again, I kept it as simple as possible for MVP. The code block below is all that is required to get a NATS server running with Containerlab. If you have [NATS Server](https://docs.nats.io/running-a-nats-service/introduction/running) installed, all you need to get a server running is the following command: `nats-server` 😊.

![The NATS Server](/blog/images/ncm/nats-server.svg)

```yaml
topology:
  nodes:
    nats:
      kind: linux
      image: nats:latest
      ports:
        - 4222:4222
```

### The NATS Runners or Minions

The NATS runners are where we get more complex. The runners are more containers. Just like the Server is running as a container, so will the runners. The runners themselves contain a simple Python application that is subscribed to specific subjects. The block below is our Dockerfile definition for building the runners.

```Docker
FROM python:3.12-alpine

WORKDIR /app

COPY requirements.txt .

RUN pip install --no-cache-dir -r requirements.txt

COPY config-replace.py .

CMD ["python", "config-replace.py"]

```

Again, trying to keep things as contained as possible. We start with a reasonably small Python base image. We then copy the requirements for our application. Once copied, we immediately install the requirements.

```shell
aiofiles
napalm
nats-py
watchfiles

```

We then copy over our `config-replace.py` file. This file holds the subscriber code and configuration replacement task for our runners. We also specify the command to run once the container is started.

```python
import asyncio
import os

import nats
from napalm import get_network_driver

# Load DEVICE environment variable to build device construct
DEVICE = os.environ["DEVICE"]

device = {
    "hostname": DEVICE,
    "username": "admin",
    "password": "admin",
}


async def send_config_replace(data):
    driver = get_network_driver("eos")
    conn = driver(**device)
    try:
        print(f"Connecting to {device['hostname']}...")
        conn.open()  # Open the connection to the device
        print("Connection established.")

        # Load and preview the replacement configuration
        print("Loading configuration...")
        conn.load_replace_candidate(config=data)
        print("Configuration changes preview:")
        print(conn.compare_config())

        # Commit the changes
        if conn.compare_config():
            print("Applying configuration...")
            conn.commit_config()
            print("Configuration replaced successfully.")
        else:
            print("No changes detected. Skipping commit.")

    except Exception as e:
        print(f"An error occurred: {e}")
        # Rollback in case of any failure
        print("Rolling back configuration...")
        conn.rollback()
    finally:
        # Close the connection
        conn.close()
        print(f"Disconnected from {device['hostname']}.")


async def main():
    nc = await nats.connect("nats://nats:4222")

    async def message_handler(msg):
        subject = msg.subject
        data = msg.data.decode()
        print(f"Received message on subject '{subject}':\n{data}")
        await send_config_replace(data)

    await nc.subscribe(f"dc1.{DEVICE}.configure", cb=message_handler)
    # Wait indefinitely until the program is interrupted
    await asyncio.Event().wait()


if __name__ == "__main__":
    asyncio.run(main())

```

That's neat, but a lot of it is just some [NAPALM](https://github.com/napalm-automation/napalm) code to perform a configuration replacement on our devices. There are two interesting points to mention. When we start these containers, we will pass along an environment variable to tell the runner which device it's effectively assigned to. We use that to build the device dictionary.

```python
DEVICE = os.environ["DEVICE"]

device = {
    "hostname": DEVICE,
    "username": "admin",
    "password": "admin",
}
```

Then, in our main function, we initially connect to our NATS Server on `nats:4222`. We also build a message handler to instruct the application to do something once a message is received on a particular subject. In our case, we will call the `send_config_replace` function while also passing along the configuration data.

```python
async def main():
    nc = await nats.connect("nats://nats:4222")

    async def message_handler(msg):
        subject = msg.subject
        data = msg.data.decode()
        print(f"Received message on subject '{subject}':\n{data}")
        await send_config_replace(data)

    await nc.subscribe(f"dc1.{DEVICE}.configure", cb=message_handler)
    # Wait indefinitely until the program is interrupted
    await asyncio.Event().wait()
```

Once we have all of these pieces defined, we can build the local container image by running the following:

```shell
docker build -t nats-config:latest .
```

The code block below is how we define the runners in our topology file and pass along their respective node assignments.

```yaml
topology:
  nodes:
...
    runner1:
      kind: linux
      image: nats-config:latest
      env:
        DEVICE: spine1
    runner2:
      kind: linux
      image: nats-config:latest
      env:
        DEVICE: leaf1
    runner3:
      kind: linux
      image: nats-config:latest
      env:
        DEVICE: leaf2
```

![The runners](/blog/images/ncm/minions.svg)

## Watch it Live

One thing I did not capture in the clips, you need to be running the watch application while testing this out. You can do so by running `python watch.py` on the builder. The clips below are woefully small, and I apologize in advance. The first clip shows a few things from left to right:

- AVD data models have been updated, and we are rebuilding the configuration file artifacts
- Localhost is running `nats sub "dc1.*.configure"` so we can view the messages (giant wall of configuration)
- Viewing `watch show ip name-server` from leaf1
- Viewing `watch show ip name-server` from leaf2
- Viewing `watch show ip name-server` from spine1

![Application in action](https://media.giphy.com/media/Z2rv4mcOHdZn59XopY/giphy.gif)

In the second clip, we get a view from within one of the runners—in this case, the runner assigned to leaf1. Once we run a new build, we can see the print statements from the config-replace script.

![Application in action](https://media.giphy.com/media/1XMByX7XL5QKzPJ03n/giphy.gif)

## Outro and Links

With the correct control mechanisms, I could see something like this leveraged in areas where a lot of change occurs frequently. Another area where I could see this is if you are starting to run into the scale limitations of running entire jobs from one server or control node. NATS deployments can include multiple servers in a dynamic mesh. I could see an army of minions assigned to the cluster, all listening for configuration jobs.

Another area where this could be interesting is sending messages to minions to perform things like health checks or validations—something for down the road. Another point I want to emphasize is that the code you see here is nowhere near production level. The config replace portion could use a lot of love and someone who really knows how to write async stuff. Thank you all for reading this far. I hope you enjoyed it and got something out of it.

- [Featured Image by Jiayuan Lian](https://unsplash.com/photos/a-group-of-minion-figurines-sitting-on-top-of-a-counter-uoII4yXTCao)
- [Github Repository](https://github.com/JulioPDX/nats-config-minions)
- [NATS](https://docs.nats.io/)
- [AVD](https://avd.arista.com/)
- [Containerlab](https://containerlab.dev/)
- [NAPALM](https://github.com/napalm-automation/napalm)
- [watchfiles](https://github.com/samuelcolvin/watchfiles)
- [aiofiles](https://github.com/Tinche/aiofiles)
