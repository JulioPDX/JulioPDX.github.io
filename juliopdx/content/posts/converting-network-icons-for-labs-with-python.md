---
title: "Converting Network Icons for Labs With Python"
date: 2021-09-17T09:31:09-08:00
draft: false
toc: true
images:
tags:
  - Network Automation
  - Python
---

![Topo Wasif Bhatti](/blog/images/topo_wasif_bhatti.png)

## Introduction

First of all, the diagrams used in this post and repository were originally created by ecceman. Please head over to the original [repository](https://github.com/ecceman/affinity) and show some love by giving it a star! When going through studies or testing a solution, I tend to lab a lot. On top of that I usually try and make my labs and topologies look pretty. I like to tell myself if the topology looks good, then I will learn more effectively. Example of the icons are above in the featured image.

I found eccemans repository after watching this video from [The Network Berg](https://www.youtube.com/watch?v=vUWjkEHPL3A&ab_channel=TheNetworkBerg). Check out the video as they demonstrate the proper folder location where images can be uploaded on an EVE-NG server. I’ll also post a small example in this post. The original files are in SVG format, whereas we need PNG files for EVE-NG.

## Manual Conversion

At first the workflow would be something like the following:

- Clone the repository
- Convert image from SVG to PNG
- Right size image to 52×52

Imagine repeating this process 10 times or even a couple hundred times. You can see how this would be pretty annoying after a while. I recall someone mentioning to me once, the best way to learn a programming language or any tool is to actually put it to use. Force the mind into a real world problem and use what has been learned to develop a solution.

## Automated Conversion

I’ll be completely honest, some of the libraries used were a first for me. But that points to the power of programming languages. Once you learn the basics, with a bit of reading and examples, one can develop very strong solutions. I’ll try and walkthrough the process of developing a simple python script that became really helpful for me.

### Research

I started where most of us start. Just heading to a web browser and typing “how to convert SVG to PNG using python”. While there were a lot of decent examples, folks new to programming may find some of them overwhelming. Luckily for you and I, stack overflow is a thing and the community usually post awesome answers to questions like the one I had. If you would like to check out the post on stack, check it out [here](https://stackoverflow.com/questions/51450134/how-to-convert-svg-to-png-or-jpeg-in-python)!

### libvips and pyvips

I wont go too much into detail on these libraries. The short of it is, libvips is an image processing library that can do a heck of a lot more than what I’m using it for. If you would like to check out their docs, [click here](https://www.libvips.org/API/current/). I like to think of pyvips as an entry point that leverages libvips under the hood. Write program in python and then it will call functions from libvips to execute.

```shell
sudo apt-get install libvips
pip3 install pyvips
```

### Interating Over Folders and Files

Now that I had tools that looked promising for our end goal, I needed a way to iterate over all folders and SVG files within them. Initially I figured the script could just create a PNG folder under the shape and color. This way users could go to svg > circle > blue > PNG, and all the PNG files would be available for circle icons that were blue. Below is an example of what I had to work with. Please note this is just a snippet.

`SVG Folder Structure`

```shell
├── svg
│   ├── circle
│   │   ├── blue
│   │   │   ├── c_bug.svg
│   │   │   ├── c_camera_blue.svg
│   │   │   ├── c_camera_dome_blue.svg
│   │   ├── gray
│   │   │   ├── c_bug.svg
│   │   │   ├── c_camera.svg
│   │   │   ├── c_camera_dome.svg
│   │   ├── green
│   │   │   ├── c_bug_green.svg
│   │   │   ├── c_camera_dome_green.svg
│   │   │   ├── c_camera_green.svg
│   │   └── red
│   │       ├── c_bug_red.svg
│   │       ├── c_camera_dome_red.svg
│   │       ├── c_camera_red.svg
│   ├── naked
│   │   ├── bug.svg
│   │   ├── camera.svg
│   │   ├── camera_dome.svg
│   └── square
│       ├── blue
│       │   ├── sq_bug_blue.svg
│       │   ├── sq_camera_blue.svg
│       │   ├── sq_camera_dome_blue.svg
│       ├── gray
│       │   ├── sq_bug.svg
│       │   ├── sq_camera.svg
│       │   ├── sq_camera_dome.svg
│       ├── green
│       │   ├── sq_bug_green.svg
│       │   ├── sq_camera_dome_green.svg
│       │   ├── sq_camera_green.svg
│
│       └── red
│           ├── sq_bug_red.svg
│           ├── sq_camera_dome_red.svg
│           ├── sq_camera_red.svg
```

Iterating over this structure initially seemed very complex but python makes this fairly simple. I leveraged [“walk” from “os”](https://www.geeksforgeeks.org/os-walk-python/). This was a bit of trial and error but when I was able to print the correct files, I could proceed with converting them. Small example below of printing root and files. I pass in a PATH variable to walk as a starting point.

```python
>>> from os import walk
>>> PATH = "svg/"
>>> for (root, dirs, files) in walk(PATH):
...     for f in files:
...         if ".svg" in f:
...             print(f"{root}/{f}")
...
svg/circle/blue/c_bug.svg
svg/circle/blue/c_camera_blue.svg
svg/circle/blue/c_camera_dome_blue.svg
svg/circle/blue/c_client_blue.svg
svg/circle/blue/c_client_vm_blue.svg
```

If you notice, root is the same for all the files shown above, “svg/circle/blue/”. All that I needed now from my initial thinking was to add a PNG folder at the end of that root. Then all the shapes and colors would have a PNG folder for the images to be converted. Please see one solution below.

`Creating PNG Folder`

```python
try:
    mkdir(path=f"{root}/png/")
except OSError as error:
    pass
```

This isn’t the best solution at all or even the most efficient. This is basically using “mkdir” to create a directory under the root and adding png. The negative here is that it will throw an error after the first time it is created. I just handle it by setting the script to continue after running into this error.

### Converting All the Things

The conversion wasnt too bad at all. I’ll explain the code upfront this time. pyvips has a function to create a thumbnail based off of a file. In this case we are looping over every single file that ends with “.svg”. We size the thumbnail as a height and width of 52. I believe these are recommended from the EVE-NG import icons guide. After that we write the file, when doing this we will place each new file under the root we are iterating over, for example “svg/circle/blue/”, appending “/png/” and ending it by changing the extension name from “svg” to “png”. Clear as mud right?

`Itsy Bitys Function`

```python
def main():

    """
    All the action
    """

    for (root, dirs, files) in walk(PATH):
        for f in files:
            if ".svg" in f:
                try:
                    mkdir(path=f"{root}/png/")
                except OSError as error:
                    pass
                convert_file = pyvips.Image.thumbnail(f"{root}/{f}", 52, height=52)
                convert_file.write_to_file(f"{root}/png/{f.replace('svg', 'png')}")
```

## Importing to EVE-NG

At this point all the SVG files have been converted to PNG format. If you would like to add these icons to EVE-NG, you can use something like [winscp](https://winscp.net/eng/index.php) or build some automation to send them to your EVE-NG server. The path you will need to store them in is “/opt/unetlab/html/images/icons/”.

## Bonus, Automating the Reference File

Something that I thought was interesting, at the time there was no overall view of every icon. Or some type of directory to check out what each icon looked like, unless you opened each file individually. Since that time ecceman has added a nice PNG that displays all the icons.

Before this existed a fellow networker also had a similar idea. Many thanks to [@pstavirs](https://twitter.com/pstavirs) for the push!

![Request from pstavirs](/blog/images/request.png)

I’ve worked with markdown before and adding image URLs. At this point we had every PNG file sized and ready to go. Why not just write a script that could build this markdown file for us. I ended up using “walk” again since I now needed to loop over every single PNG image. We decided on three icons per row, with name on the right side of the icon. I played with a bit of simple math in the loop to tell the program when to drop to the next line. If you have any questions on the process feel free to ping me. If you want to see the file in the raw [click here](https://raw.githubusercontent.com/JulioPDX/affinity/master/reference.md), crazy right? Rendered version below!

![Reference File Rendered](/blog/images/ref_mark.png)

`ref_build.py`

```python
def main():

    """
    All the action
    """

    with open("reference.md", "w") as my_file:
        # Headers and settings alignments for file
        my_file.write("# Reference for Icons\n")
        my_file.write("\n")
        my_file.write("|Icon|Name|Icon|Name|Icon|Name|\n")
        my_file.write("|:---:|:---:|:---:|:---:|:---:|:---:|\n")
        for (root, dirs, files) in walk(PATH):
            count = 0
            # Iterating over all files with .png extension
            for f in files:
                if ".png" in f:
                    # Simple math checks on matches and reset
                    # counter on third entry
                    count += 1
                    if count == 1:
                        my_file.write(f"|![{f}]({root}/{f})|{f.replace('.png', '')}|")
                    if count == 2:
                        my_file.write(f"![{f}]({root}/{f})|{f.replace('.png', '')}|")
                    if count == 3:
                        my_file.write(f"![{f}]({root}/{f})|{f.replace('.png', '')}|\n")
                        count = 0
        my_file.write("\n")
```

## Outro and Links

Thank you all for reading this far, it really means a lot. This was probably one of the most asked questions I get, “How do you get those icons in your topology?”. If you would like to add these images to your EVE instance, feel free to clone down the repository and add whatever you like. Putting what you learn to use has many advantages and this one was a lot of fun. Shout out to [Jordi](https://twitter.com/Network_101010), Wasif, and [Srivats](https://twitter.com/pstavirs). Thank you for being so positive in the community! I wish everyone all the best on your future studies and happy labbing!

![Happy Community](/blog/images/happy_community.png)

- [Original repository from ecceman, amazing!](https://github.com/ecceman/affinity)
- [Fork of original repo to add conversion script and reference builder](https://github.com/JulioPDX/affinity)
- Featured Image credit to Wasif Bhatti, thank you!
