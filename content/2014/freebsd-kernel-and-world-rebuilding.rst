=====================================
 FreeBSD Kernel and World Rebuilding
=====================================

:date: 2014-12-08 02:05
:modified: 2014-12-09 17:11
:tags: sa, freebsd, kernel, driver
:category: memo
:slug: freebsd-kernel-and-world-rebuilding
:authors: Zespre Schmidt
:about_author: Bug generator
:email: starbops@gmail.com
:summary: System Administration Kernel and Driver Note

Kernel-wide Design Approaches
=============================

Monolithic Kernel
-----------------
- Entire operating system is working in kernel space
    - Efficient
    - But hard to maintain
- Can dynamically load executable modules at runtime
- Linux, \*BSD, MS-DOS, Windows 9x series

Micro Kernel
------------

- Address space management, inter-process communication, scheduling
- Mach, QNX, L4

Hybrid Kernel
-------------

- Monolithic + Micro
- MacOS, Windows NT series, DragonFly BSD

Why Building Custom Kernel
==========================

Building a custom kernel is often a rite of passage for advanced FreeBSD users.
The procedure, however, is time consuming, can provide benefites to the FreeBSD
system. Unlike ``GENERIC`` kernel provided by the default FreeBSD system aims to
be generic, it must support a very wide range of hardware. A customized kernel
can be stirpped down to fit your needs. The advantages are listed below:

- Faster boot time
- Lower memory usage
- Additional hardware support

Finding The System Hardware
===========================

If your FreeBSD is dual boot with Windows, maybe Windows device manager can help
you out. Otherwise, use the following methods can list the hardware which are
currently in your box.

- ``pciconf -l``
- ``cat /var/log/dmesg.boot``

The Configuration File
======================

If ``/usr/src`` does not exist or it is empty, download it using Subversion:

.. code-block:: bash

    # svn checkout https://svn0.us-west.FreeBSD.org/base/release/9.2.0 /usr/src

The sample kernel configuration file resides in ``/usr/src/sys/<arch>/conf``
sub-directory, the file named ``GENERIC`` being the one used to build your
initial installation kernel.

Do not make edits to ``GENERIC`` directly. Also, we will keep the kernel
configuration file elsewhere and create a symbolic link to the file.

.. code-block:: bash

    # cd /usr/src/sys/amd64/conf
    # mkdir /root/kernels
    # cp GENERIC /root/kernels/MYKERNEL
    # ln -s /root/kernels/MYKERNEL

References for Configuring Your Own Kernel
------------------------------------------

The file NOTES contains entries and documentation for all possible devices, not
just those commonly used. It is the successor of the ancient LINT file, but in
contrast to LINT, it is not buildable as a kernel but a pure reference and
documentation file.

To build a file which contains all available options:

.. code-block:: bash

    # cd /usr/src/sys/amd64/conf
    # make LINT

Reminder: When in doubt, just leave support in the kernel.

Building and Installing a Custom Kernel
=======================================

First change directory into ``/usr/src`` then compile the custom kernel by
specifying the configuration file:

.. code-block:: bash

    # cd /usr/src
    # make buildkernel KERNCONF=MYKERNEL

Install the compiled kernel into ``/boot/kernel/kernel``. The old kernel will be
moved to ``/boot/kernel.old/kernel``:

.. code-block:: bash

    # make installkernel KERNCONF=MYKERNEL

By default, all kernel modules are rebuilt when a custom kernel is compiled. To
compile the kernel faster, edit the ``/etc/make.conf`` before starting to build
the kernel:

.. code-block:: text

    MODULES_OVERRIDE = linux acpi

Alternately, this lists the modules which are excluded from the build process:

.. code-block:: text

    WITHOUT_MODULES = linux acpi sound

Troubleshooting and Failover
============================

Config Fails
------------

If ``config`` fails, it will show up an error message along with the line of the
error configuration. Try to fix it by comparing your configuration to
``GENERIC`` or ``NOTES``.

.. code-block:: text

    config: line 17: syntax error

Make Fails
----------

Sometimes the error is not severe enough to be caught by ``config``, then
``make`` will failed. Send an email to the FreeBSD general questions mailing list
with the kernel configuration file attached, if everything in the configuration
file seems right.

The Kernel Does Not Boot
------------------------

    - ``/var/log/messages``
    - ``dmesg``

Make sure to keep a copy of ``GENERIC``, or some kernel that is known to work.
Otherwise the ``/boot/kernel/kernel.old`` will be overwritten with the last
installed kernel, which may not be bootable.

.. code-block:: bash

    # mv /boot/kernel /boot/kernel.bad
    # mv /boot/kernel.good /boot/kernel

The Kernel Works, But Some Utilities Do Not
-------------------------------------------

After successfully compiled the custom kernel and rebooted the
machine, everything looks well except some utilities do not work. This is
because the kernel version differs from the one that the system utilities have
been built with. For example, a kernel built from -CURRENT source is installed
on a -RELEASE system, many system status command, e.g., ``ps``, ``vmstat`` will
not work.

So the only way to use them as usual is to "recompile and install a world" built
with the same version of the source tree as the kernel.

Rebuilding World
================

There might still contain old object files that generated in earlier builds. To
minimize the potential problem, it is good to remove them.

.. code-block:: bash

    # cd /usr/obj
    # chflags -R noschg *
    # rm -rf *

Compile the new compiler and other tools, then use the new compiler to compile
the rest of the new world.

.. code-block:: bash

    # cd /usr/src
    # make buildworld

Use the new compiler residing in ``/usr/obj``` to build the new kernel.

.. code-block:: bash

    # make buildkernel KERNCONF=MYKERNEL

Install new kernel and kernel modules.

.. code-block:: bash

    # make installkernel KERNCONF=MYKERNEL

Drop the system into single-user mode to reduce the problems cause by multiple
users environment. Also, most of the services will be shut down for the same
purpose.

.. code-block:: bash

    # shutdown now

Once in single-user mode, run these commands if the system is formatted with
UFS.

.. code-block:: bash

    # mount -u /
    # mount -a -t ufs
    # swapon -a

Or instead of UFS, ZFS is used:

.. code-block:: bash

    # zfs set readonly=off zRoot
    # zfs mount -a

If the keyboard binding is going to be changed, configure it now.

.. code-block:: bash

    # kbdmap

If the CMOS clock is set to localtime, run the following command. To check
wether if the clock is set to localtime or not, there is a quick command
``date`` to examine.

.. code-block:: bash

    # adjkerntz -i

Rebuilding the world will not update certain directories such as ``/etc``,
``/var`` and ``/usr``. The Bourne shell script ``mergemaster`` will determine
the difference between the files in ``/etc`` and ``/usr/src/etc``. You will have
four choices for each file that differs:

- Delete the new file
- Install the new file
- Merge the new file with the file that currently installed
- View the result again

.. code-block:: bash

    # mergemaster -p

Install the new world and system binaries from ``/usr/obj``.

.. code-block:: bash

    # cd /usr/src
    # make installworld

Update any remaining configuration files.

.. code-block:: bash

    # mergemaster -iF

Delete any obsolete files. Otherwise they might cause problems.

.. code-block:: bash

    # make delete-old

A reboot is needed to load the new kernel and new world with the new
configuration files.

.. code-block:: bash

    # reboot

Remove obsolete libraries. Old libraries might have security or stability
issues. Make sure that all installed ports are rebuilt.

.. code-block:: bash

    # portmaster -af
    # make delete-old-libs

References
==========

- `Chapter 9. Configuring the FreeBSD Kernel`__
- `Chapter 24.6. Rebuilding World`__

.. __: http://www.freebsd.org/doc/en/books/handbook/kernelconfig.html
.. __: http://www.freebsd.org/doc/en/books/handbook/makeworld.html

