---
layout: post
title: Removing Inaccessible VirtualBox Virtual Machines
description: This is a quick way to remove inaccessible virtual machines from Virtual Box.
---

This is a quick way to remove inaccessible virtual machines from Virtual Box, without using the VirtualBox GUI.

Sometimes you might receive the following error when attempting to SSH into a VM on your machine:

    $ homestead ssh
    Your VM has become "inaccessible." Unfortunately, this is a critical error
    with VirtualBox that Vagrant can not cleanly recover from. Please open VirtualBox
    and clear out your inaccessible virtual machines or find a way to fix
    them.

The error message suggests you open the VirtualBox GUI and "clear" the affected machines. However, sometimes this doesn't work as expected, so here's a quick way to resolve the issue from the terminal:

    $ VBoxManage list vms

You'll receive a list of all virtual machines available, along with their GUID:

    "machine-1" {18daeed5-2be7-4e7f-a36a-b8b8e27a9f11}
    "machine-2" {564aa0b3-a856-4b9a-9589-aed44a6c143b}

Now you can run the following command against a chosen VM:

    $ VBoxManage unregistervm [GUID]

No need to use the GUI, this way is quicker and easier!
