Test your bootable USB drive with QEMU
======================================

:author: Pascal Hartig
:authorrank: https://plus.google.com/102129173185887281562
:date: 2012-09-09
:summary:
    Creating a bootable USB drive can be tricky at times. With QEMU you can
    check whether your USB drive is bootable without rebooting your computer or
    trying it on a different system.

Creating a bootable USB drive can be tricky at times. With QEMU you can check
whether your USB drive is bootable without rebooting your computer or trying it
on a different system.

Find The Device
---------------

As a first step, you need to find the bus and device IDs of your USB drive. To
get those, run ``lsusb`` and look for your device::

    Bus 001 Device 008: ID 0781:5151 SanDisk Corp. Cruzer Micro Flash Drive

In this case, the bus ID is *001* and the device id *008*.

Start QEMU
----------

Now all you have to do is run QEMU with the two parameters we just found::

    sudo qemu-system-x86_64 -m 512 -enable-kvm -usb -device usb-host,hostbus=1,hostaddr=8

You need to run this with root privilegues, because QEMU needs access to the
corresponding files under ``/dev/bus/usb/``.

That's it. You will now either see your system boot from USB or get an error.
