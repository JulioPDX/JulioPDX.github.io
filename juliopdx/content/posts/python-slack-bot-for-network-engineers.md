---
title: "Python Slack Bot for Network Engineers"
date: 2021-05-24T10:19:32-08:00
draft: false
toc: true
images:
  - /blog/images/jason-leung.jpg
tags:
  - Python
---

## Introduction

Hello and thank you for joining me in another blog post. I’ve wanted to mess with getting a bot running on Slack for a while now. After coming across a fantastic post by Mason Egger at Digital Ocean (linked at the end), I figured now is as good a time as any. Think of that post as a prerequisite to get you started before following along with this one.

## Caveats

A few caveats I should include. I’m a fairly novice python user, expect to see areas where code can be refactored and even total rewrites that would make the organization better. What I will demonstrate in this post is what made sense to me at this point in time. I hope you enjoy and maybe get inspired to build something you can use to assist you in your daily workflow.

## Network Engineering Bot

Below are a few of the goals I had when creating this bot.

- Must be multi-vendor
- Ability to call different (non static) devices
- Bot should react to user messages

Lets walk through some code and I hope to answer how those goals were met. I’ll try not to repeat information that was already shared in Masons post. The initial code from Masons example has you create a class called CoinBot. It only made sense to me to create a separate class to define our network engineering bot. In this case we will call it NetBot, but you can name your bot whatever you like.

`get_network.py`

```python
import json
from napalm import get_network_driver
from yaml import safe_load
import urllib3

# Disable warnings
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

class NetBot:
    """
    Defining a class called NetBot
    """

    USERNAME = "admin"
    PASSWORD = "admin"
    NET_BLOCK = {
        "type": "section",
        "text": {
            "type": "mrkdwn",
            "text": (
                "Getting interface information for device :slightly_smiling_face:"
            ),
        },
    }

    def __init__(self, channel, device_name):
        """
        constructor for class
        """
        self.channel = channel
        self.device_name = device_name
```

A few things to note. The variables you see in all caps are called class attributes/constants. Essentially things in the program that wont change. I kept it simple and created some credentials for authentication as class attributes. In reality this would be an environment variable or pulled from a secure source. The NET_BLOCK class attribute is just there so the bot can respond with some generic message to the user.

One of the first things that came to mind when building the bot is how will users know how to run the bot? I created the following method to solve that little problem. We are basically reading the readme file in our root directory and then returning all the data required to compose a message in slack. The following is added under our NetBot class from above.

`get_network.py`

```python
    def send_help(self):
        """
        Sending all the help
        """
        with open("README.md", "r") as reader:
            readme = reader.read()
        return {
            "channels": self.channel,
            "filename": "README.md",
            "filetype": "markdown",
            "content": readme,
        }
```

![netbot help](/blog/images/netbot_help.png)

## Interacting With Network Devices

I mentioned before I wanted this to be multi-vendor. In this case we are going to use the popular NAPALM library!

```shell
pip install napalm
pip install pyaoscx
pip install napalm-aruba-cx
```

In theory you can use whatever you are comfortable with; Netmiko, NAPALM, Nornir, or Ansible. As long as the end result returns data in a format we can use to craft a Slack message. For our networking example, we will use NAPALM to retrieve the network interfaces from two devices. One being a Cisco IOL router and the other an Aruba CX switch. Below you will see the two methods used to put this all together. The first is pretty standard syntax to connect to device, run the get_interfaces function, and return as pretty JSON.

Quick deviation, I almost forgot to mention the inventory. In a grander scale, you would most likely have some dynamic inventory that is cached to the app running or the code actually interacts with your inventory source to pull the correct information. Think Netbox API call to get device information for a device named XYZ. This would then have management IP, platform, and whatever else would be needed to connect using NAPALM. In our case we are using a very simple YAML file with the two hosts mentioned earlier.

`hosts.yaml`

```yaml

---
R1:
  name: R1
  platform: ios
  mgmt: "192.168.10.168"
ArubaCX:
  name: ArubaCX
  platform: aoscx
  mgmt: "192.168.10.169"
```

As promised, here is the code that interacts with our network devices.

`get_network.py`

```python
    def _get_facts(self):
        """
        Method that will retrieve host facts
        """
        with open("hosts.yaml", "r") as handle:
            host_root = safe_load(handle)

        driver = get_network_driver(host_root[f"{self.device_name}"]["platform"])
        conn = driver(
            hostname=host_root[f"{self.device_name}"]["mgmt"],
            username=self.USERNAME,
            password=self.PASSWORD,
        )
        conn.open()
        facts = conn.get_interfaces()
        my_file = json.dumps(facts, indent=2)
        return my_file
```

The second portion of interacting with the networking device is constructing the message. The following method will return a working format to post a file type message in Slack. Notice towards the bottom the “content” key is actually calling the “_get_facts” method above? The _get_facts method is just returning some pretty formatted JSON string that we can then pass into our method to build a slack message.

`get_network.py`

```python
    def get_file_payload(self):
        """
        Method used to post files from data gathered on device
        """
        return {
            "channels": self.channel,
            "filetype": "javascript",
            "content": self._get_facts(),
            "filename": f"{self.device_name}-interfaces.json",
        }
```

## Running the Application

As mentioned previously, this code is heavily borrowed from Masons post! So much so that I left in the CoinBot learnings because it is an amazing reference and the code comments are great for learners. Ill cut out some of the extra information to just focus on the action, full code is available on my GitHub (linked below).

`app.py`

```python
import os
import logging
from flask import Flask
from slack import WebClient
from slackeventsapi import SlackEventAdapter
from coinbot import CoinBot
from get_network import NetBot

# Initialize a Flask app to host the events adapter
app = Flask(__name__)

# Create an events adapter and register it to an endpoint in the slack app for event ingestion.
slack_events_adapter = SlackEventAdapter(
    os.environ.get("SLACK_EVENTS_TOKEN"), "/slack/events", app
)

# Initialize a Web API client
slack_web_client = WebClient(token=os.environ.get("SLACK_TOKEN"))

# Define bot ID so it will not respond to itself
BOT_ID = slack_web_client.api_call("auth.test")["user_id"]
```

Lets break that code down a bit. We will be utilizing Flask to do most of the heavy lifting. The imports mentioned above will import the required Flask, Slack, and NetBot packages. We utilize two environment variables for authentication. The BOT_ID constant is used in code later on to make sure the bot does not respond to itself. The next portions actually perform the execution of all of our code. I will break down one portion to keeps things short and sweet, but most all of them follow the same workflow.

`app.py`

```python
def get_network_info(channel, device_name):
    """
    run the get_message_payload and get_file_upload method
    """
    net_bot = NetBot(channel, device_name)
    my_message = net_bot.get_message_payload()
    file_output = net_bot.get_file_payload()
    slack_web_client.chat_postMessage(**my_message)
    slack_web_client.files_upload(**file_output)
```

We are creating a function called get_network_info that takes in two parameters. When this function is executed, it will instantiate an instance of NetBot, send a generic message, and upload the file created from the get_file_payload method.

The next portion of the code is used to handle how the bot interacts with messages seen or what is executed when a certain message is seen. From part one of Masons post, we subscribed the bot to events (and we filter on messages). Once we get a message we strip out certain portions that can then be reused to respond in the proper channel, react to a specific message (timestamp), and make sure the bot does not respond to itself.

`app.py`

```python
# When a 'message' event is detected by the events adapter, forward that payload
# to this function.
@slack_events_adapter.on("message")
def message(payload):
    """
    Parse the message event, and if the activation string is in the text,
    simulate something and send result
    """

    # Get various portions of message
    event = payload.get("event", {})
    text = event.get("text")
    user_id = event.get("user")
    timestamp = event.get("ts")
    channel_id = event.get("channel")

    # Making sure the bot doesnt respond to itself
    if BOT_ID != user_id:
```

The next piece of code is used to parse the messages that are seen by the bot. If a string matches one of the if clauses, some code will be executed. In this case we are looking for “netbot get network interfaces”. If you noticed in the readme, we are asking users to enter “netbot get network interfaces device=something”. The code will parse this output and strip “device=” to get the final device name. This will then be passed into the get_network_info function and execute the program.

`app.py`

```python
        if "netbot get network interfaces" in text.lower():
            full_text = text.split()
            device = full_text[-1]
            my_device = device.replace("device=", "")
            slack_web_client.reactions_add(
                channel=channel_id, name="robot_face", timestamp=timestamp
            )
            slack_web_client.reactions_add(
                channel=channel_id, name="rocket", timestamp=timestamp
            )
            return get_network_info(channel_id, device_name=my_device)
```

You may have noticed a few commands the are prepend with “slack_web_client.reactions_add”. I wanted a way for the bot to respond to a user inputting a message that triggers the bot. Almost like a confirmation of message received. This can be message specific as I have made it for “netbot help” or “netbot get network interfaces”. Remember just a bit ago how we strip certain information from the message received like channel and timestamp ID? This is used throughout the code but here we are statically setting the types of reactions executed for this match. In the case of “get_network_info” the bot will react with a robot face and rocket.

![Get Interfaces](/blog/images/get_interfaces.png)

The rest of the code is boilerplate to execute the application and enable logging. No changes from the original post at Digital Ocean. I want to thank Mason for the amazing blog post to kick start this spark for me. Thank you for reading this far, really means a lot! I hope what I’ve written makes a bit of sense and maybe inspires you to create something awesome.

## Bugs/Issues

- Bot reruns get network interface code even when new message is not sent (not seen when reactions are used).
- When reaction is used multiple runs are not seen but app reports error 500 (already reacted).

## Links

- [Featured Image by Jason Leung](https://unsplash.com/photos/HBGYvOKXu8A)
- [How To Build a Slackbot in Python on Ubuntu 20.04](https://www.digitalocean.com/community/tutorials/how-to-build-a-slackbot-in-python-on-ubuntu-20-04)
- [Python Slack Bot Tutorial Playlist by Tech With Tim](https://www.youtube.com/playlist?list=PLzMcBGfZo4-kqyzTzJWCV6lyK-ZMYECDc)
- [GitHub Repository](https://github.com/JulioPDX/ne_bot_example)
