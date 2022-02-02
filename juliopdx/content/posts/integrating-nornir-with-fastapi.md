---
title: "Integrating Nornir With FastAPI"
date: 2021-09-01T18:52:28-08:00
draft: false
toc: true
images:
tags:
  - Python
  - Nornir
  - FastAPI
  - Cisco
  - Aruba
  - Arista
  - Network Automation
---

## Introduction

I recently saw a fellow engineer share an amazing article on [Real Python](https://realpython.com/fastapi-python-web-apis/). It was a post by Sebastián Ramírez, the creator of FastAPI. I’m fairly comfortable working with APIs but I’ve never even thought about making one. The prospect seems so daunting, especially when I’m still trying to master the foundational skills of python. FastAPI does a ton of heavy lifting to make this process incredibly easy to get started and build web APIs.

When learning something new I try and incorporate it to something that interest me. A decent amount of the time it relates to network engineering. Another example could be learning how to interact with a [Pokemon API](https://pokeapi.co/). I wanted to see if I could possibly integrate Nornir with FastAPI.

## Objective

My initial thinking for this experiment was to use Nornir and NAPALM to connect to devices and retrieve information. If this could then be linked to separate URLs, that would make for a pretty slick way of interacting with networking devices. I really thought I would be making some script that was much much longer than what I actually implemented.

I wont go too deep into Nornir or FastAPI. The Real Python article linked is an amazing resource as well as the FastAPI documentation. I’ve written about Nornir before, and if you are curious about the structure, feel free to check out one of my previous [posts](https://juliopdx.com/2021/02/27/network-validation-with-nornir-napalm/).

## Topology

![EVE Topo Multi](/blog/images/eve_topo_multi.png)

The topology consists of three networking nodes from three popular vendors. I tried to include a variety to show how automation can remove some thought when trying to return data from devices. Whether its vendor A or vendor B, at the end of the day we want the information required for the task at hand.

## Script Breakdown

`All the Imports`

```python
from fastapi import FastAPI
from nornir import InitNornir
from nornir_napalm.plugins.tasks import napalm_get
from yaml import safe_load
import urllib3
```

- FastAPI – Used to create a class instance of FastAPI.
- InitNornir – Similar to FastAPI above, in this case Nornir will be initialized with a config file.
- napalm_get – Heavily used to retrieve all the information in our API calls.
- safe_load – Will be used to load hosts.yaml file (specific API call).
- urllib3 – At the moment this will simply be disabling insecure warnings in the terminal output.

`Inits`

```python
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

nr = InitNornir(config_file="config/config.yaml")

app = FastAPI()


@app.get("/")
async def root():
    """Says hi!"""
    return {"message": "Hello JulioPDX"}
```

As mentioned previously, we will be disabling the insecure warnings for our lab. We are initializing Nornir with the nr variable and FastAPI to the app variable. Something new is the ‘@app.get(“/”)’ on line eight. From the article this is known as a decorator in python and it relates to the function definition immediately below it. You will see this pattern further in this writing. For this example, if we run the app pointing to “/” it should return “Hello JulioPDX”. Lets start the app!

`Starting the App`

```shell
(venv) [juliopdx@pbpro net_auto_fastapi]$ uvicorn play:app --reload
INFO:     Will watch for changes in these directories: ['/home/juliopdx/git/net_auto_fastapi']
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [99179] using watchgod
INFO:     Started server process [99184]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

Sticking close to the learning so far, uvicorn is used as our server to run the app. The reload option will restart the service anytime a change is made to our play.py file and saved. Lets check out the path mentioned in earlier code.

![Looking good!](/blog/images/fastapi_hello.png)

Now that I have my local environment to the point of the original article, I wanted to start with something simple to test my knowledge. Could there be a way to let users know what all devices will be loaded and available in Nornir? Check out the code below.

`Get All the Things`

```python
@app.get("/devices")
async def get_devices():
    """Returns a list of devices loaded from our hosts file"""
    with open("./config/hosts.yaml", encoding="utf-8") as file:
        devices = safe_load(file)
    return {"devices": devices}
```

This is pretty neat, its a simple python operation to open a file and set the contents to a variable. Incorporating the output into FastAPI was incredibly easy. This will also show a user what devices they can work with at a simple URL or endpoint. Something else to note is that the credentials are not exposed to the user, only the portions within the hosts.yaml file.

![Devices](/blog/images/fastapi_devices.png)

This next portion was the most interesting to me. NAPALM has a concept called getters. Basically these functions will go and grab different pieces of information from a device. I initially thought I would have to write a function for every getter. Luckily for me and you viewers watching at home, we can use path parameters when defining these endpoints! For example, if I wanted to interact with the ArubaCX device and the facts getter, I would need to provide two parameters to the endpoint. This allows the operator to just enter the URL with the correct device name and getter, all in one function definition!

`All the Getters`

```python
@app.get("/devices/{hostname}/napalm_get/{getter}")
async def get_config(hostname: str, getter: str):
    """Function used to interact with NAPALM and devices"""
    rtr = nr.filter(name=f"{hostname}")
    return rtr.run(name=f"Get {hostname} {getter}", task=napalm_get, getters=[f"{getter}"])
```

|                                                                      |                                                              |
| :------------------------------------------------------------------: | :----------------------------------------------------------: |
|       ![NAPALM Arista ARP](/blog/images/napalm_arista_arp.png)       | ![NAPALM Arista Facts](/blog/images/napalm_arista_facts.png) |
|       ![NAPALM Aruba LLDP](/blog/images/napalm_aruba_lldp.png)       |  ![NAPALM Aruba Facts](/blog/images/napalm_aruba_facts.png)  |
| ![NAPALM Cisco Interfaces](/blog/images/napalm_cisco_interfaces.png) |  ![NAPALM Cisco Facts](/blog/images/napalm_cisco_facts.png)  |

## Outro and Links

Thank you for reading this far, really means a lot. So much of this can be improved and I’m excited to keep learning about python and FastAPI. Please check out the links below for more information.

- [Real Python - Using FastAPI to Build Python Web APIs](https://realpython.com/fastapi-python-web-apis/)
- [Nornir Documentation](https://nornir.readthedocs.io/en/latest/)
- [NAPALM Getters](https://napalm.readthedocs.io/en/latest/support/#getters-support-matrix)
- [Repository for code](https://github.com/JulioPDX/net_auto_fastapi)
