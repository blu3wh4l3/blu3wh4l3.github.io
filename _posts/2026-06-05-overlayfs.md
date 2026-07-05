---
title: OverlayFS - Copy-up to Privilege Escalation(CVE-2023-0386)
date: 2026-06-05 13:57:00 +/-TTTT
categories: [CVE, PRIVILEGE ESCALATION, VULNERABILITY]
tags: [linux, priv_esc, cve]     # TAG names should always be lowercase
---

Hey everyone, welcome to **Bluewh4l3 Byt3s!** Technically, this is my second blog post—my very first was a TryHackMe writeup over on Medium a long time ago—but this marks the official launch of this site. Today, we’re diving deep into OverlayFS and a critical CVE tied to it. We will be breaking down how it works under the hood and how it can be exploited, so I hope you find it valuable. Let’s jump right in!

> Before jumping into the CVE, we first need to understand whats overlayFS and how does it work and what makes it vulnerable. so bear with me for sometime. I'll give you a clear background information on overlayFS, so you'll understand the CVE much clearly.
PS: It's gonna be a bit long.
{: .prompt-info }

## What's OverlayFS?
OverlayFS stands for Overlay File System. overlayFS is a feature of linux kernel that combines two filesystems(directories)  and shows it as a single unified view to the user. It gives an illusion of a single filesystem to the user but under the hood it combines two filesystems - upper and lower and shows it as a single one. 

![tranpsarent-sheet-example](/assets/img/transparentsheet.png)

- Think of overlayFS as a combined view when we put a transparent sheet on top of a drawing, through this sheet we can see the entire drawing.
- We cannot change the drawing but for a person viewing through the sheet, they can add additional things by drawing on the transparent sheet without making any changes on the original drawing and hence the original drawing becomes like a read-only thing.
- To the end user, the additional changes along with the base layer gives a combined view.
- Now lets look at the components of overlayFS based on this analogy:

lower layer - The base drawing

upper layer - The transparent sheet

merged view(overlayFS) - The combined view when someone looks through both the layers

## OverlayFS in action - How Docker uses it
The best example of an overlayFS are docker containers. Lets look at how docker containers work
- Lets say we wanna pull an ubuntu image from docker, when we run docker pull command it'll first check if the image is available locally if not it'll pull an image from the docker hub.
- Now this docker image is the lower layer. With this image we can create multiple containers. This is where overlayFS comes into play.
- When multiple containers are created and user views each containers, they'll be viewing a single base image of ubuntu which is mostly stored locally at ```/var/lib/docker/``` in linux
- It gives an illusion that user is viewing an entire ubuntu filesystem of its own.
- When user wants to make any changes in the container(merged view), then that change is recorded in a separate directory: ```/var/lib/docker/overlay2/container-cache-id/diff/``` but the base image of ubuntu never changes. It remains same across all the containers.


## Copy-up operation in overlayFS
Now the important thing that comes is the copy-up operation in overlayFS. Like I explained earlier, when a user makes any changes in the merged view, that change is stored in another directory. Now changes can be of two types: something new is created in the merged view which does not exists in the lower layer or making edits to something which exists in the lower layer.
- As we already know we cannot directly edit anything in the lower layer, so when user tries to make any edits to any file which already exists in the lower layer, kernel triggers a copy-up operation.
- Meaning, whichever file user tries to edit, kernel makes a copy of that file and stores in the upper layer.
- User works on the copy of the file rather than the original one in the lower layer.
- Copy-up is only triggered when user tries to modify an existing file, no copy-up is triggered for read operation or a new file creation.

This is what a copy-up operation looks like.

### Metadata and Permission Copy-up
