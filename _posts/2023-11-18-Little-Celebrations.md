---
title: Little celebrations for remote workers
description: We all deserve sharing our joy about our little accomplishments!
date: 2023-11-18 15:40:00 +0200
categories: []
tags: [toy-project,script,shell]
---

## Idea

There are lots of people nowadays working remotely through their laptops in some quiet place within their home office. 🧑‍💻 While most of the time, we achieve great accomplishments, there are occasions when we make a great impact on our company, colleagues, or community through these achievements.

This is a great occasion to celebrate 🎉 such moments!

Working remotely or even in an asynchronous way makes such celebrations a lonely process. So, there is no apparent and easy way to share such moments with co-workers, within a community, or with some friends.

This is where I decided to do something about it and share my solution with everyone so we can spread our joy!

> Inspired by similar tool used by [Michael Milewski](https://www.linkedin.com/in/michael-milewski/) and [Selena Small](https://www.linkedin.com/in/selenasmall/) during their keynote presentation at [Open Conf](https://www.open-conf.gr/) 2023.

## Requirements

So, we need a way to record our celebrations and then share them with our peers.

As we say, 'A picture is worth a thousand words,' so what's better than a picture to share our joy? ...

Correct: a series of pictures - this is what we want to record and share with other peers for these special cases! 🎞️

The purpose is to record a series of images in a proper format, suitable for sharing, only upon the successful completion of a command in our shell. 🖥️
Gif file, compared to video, seems a more appealing solution as it's easier to get shared and hosted through different services and tools.

So, let's say that we wanted to celebrate upon successful execution of my application, we would like to run something like:
```shell
$ celebrate my_application
```

or for commiting my source code:
```shell
$ celebrate git commit -m "Adds missing documention"
```

Upon successful completion, a recording process should begin to capture and showcase our joy for achieving such a great accomplishment! 😎

## Implemantation

To sum things up, we would like a bash script that allows us to execute any given command, execute it, and upon successful completion, record and store a few seconds of a GIF using a connected image-capturing device (web camera). Recording should start as soon as the given command completes its execution and last for just a few seconds – 10 seconds seem to be enough for our little celebrations!

### Prerequisites

There is a need for two distinct tools for our purpose. The first one should be capable of capturing images from an attached image-capturing device and storing them. The second one should be able to combine all these images from a given directory into a single GIF file.

For capturing images, we chose [imagesnap](https://github.com/rharder/imagesnap), which enables us to grab images from any capturing device. The second tool is a part of [ImageMagick](https://imagemagick.org/index.php) and is called `convert`. It allows us to merge all images within a folder to generate a single animated GIF file

### The Script

Now it is time to combine the tools with the process described before in order to generate a single script to allow our celebrations!

This is the used script. Let's call it `celebrate`:

```shell
#!/bin/bash

"$@"
# If command returns no error
if [ $? -eq 0 ]; then
    # Save the current directory
    current_dir=$(pwd)

    # Store to /tmp/celebrate after removing previous content
    cd /tmp
    rm -r celebrate
    mkdir celebrate
    cd celebrate
   
    # Grab 20 images with 0,1" delay between them
    echo "🎉 Now you can celebrate!"
    imagesnap -d FaceTime -t 0.1 -n 30 -w 1  > /dev/null 2>&1
    echo "Great! Share responsibly! 🎉"
    # Convert all them to a gif
    convert *.jpg celebrate.gif > /dev/null 2>&1

    # Open gif file
    qlmanage -p celebrate.gif > /dev/null 2>&1 &

    # Go back to the previous directory
    cd $current_dir
fi
```
If we add it to the `PATH` of our OS we can run it liek this:

```shell
$ celebrate git commit -m "Adds missing documention"
```
> Check the Configuration paragraph for changing the script according to your setup and needs.

 It will first run the provided command: `celebrate git commit -m "Adds missing documention"` through the 
 ```
 "$@"
 ```

If the commands is completed successfully, the script will transfer working directory to `/tmp` where it will create a new directory `celebrate` that we will store the grabbed images. Then images will be grabbed with the 
```shell
$ imagesnap -d FaceTime -t 0.1 -n 30 -w 1  > /dev/null 2>&1
```
command. 

This will grab 30 images with 0.1" delay between them and store all to the curent directory. Then the command
```shell
$ convert *.jpg celebrate.gif
```

will combine all `*.jpg` images if the current directory to create a single animated gif file with name `celebrate.gif`.

```shell
$ qlmanage -p celebrate.gif
```
will open the generated gif file for review and further usage and imediatelly return back to previous directory.

### Configuration

Provided script can be altered to be in align with someones setup and requirements.

`imagesnap` supports various devices which can be listed using:
```shell
$ imagesnap -l
Video Devices:
=> EpocCam
=> OBS Virtual Camera
=> Logitech BRIO
=> FaceTime HD Camera (Built-in)
=> USB 2.0 Camera
```

So then we can use the `-d` argument to set the name of the capturing device (partial mach can be used here).
Flag `-t` sets the delay to use between grabbing photos in seconds (Supports sub second precission using a decima numbers).
Flag `-n` sets the amount of photos image to grab. So, multiplying these two arguments will give the total duration of photo grabbing in seconds. The `-w` flag sets a warmup period to allow for device to become ready before grabbing the pgotos.

Instead of `qlmanage`, another tool for presenting the generated gif can be used, such as a browser or an image viewer application.

## Demo time

Here is a sample output of the script after succesfully commiting some code:

```shell
$ celebrate git commit -m "Adds missing documention"
🎉 Now you can celebrate!
Great! Share responsibly! 🎉
```

And the generated gif file that grabbed by the script:

![celebrate](/assets/posts/Little-Celebrations/celebrate.gif)

🎉 Hooray 🎉
