About
=====

Procszoo is a small Python module that gives you full
power to manage your processes by Linux namespaces.

## Contents
- [wiki](https://github.com/xning/procszoo/wiki)
- [Goals](#goals)
- [Requirements](#requirements)
- [Building](#building)
- [Try It](#try-it)
- [Getting Your Feet Wet with the *procszoo* Module](#getting-your-feet-wet-with-the-procszoo-module)
- [Networks](#networks)
- [Docs](#docs)
- [Known Issues](#known-issues)
- [Exported Functions and Objects](#exported-functions-and-objects)
- [Test Platforms](#test-platforms)

## Goals
--------

Procszoo aims to provide you a simple but complete tool and
you can use it as a **DSL** or an embeded programming language
which let you operate Linux namespaces by Python.

Procszoo gives a smart *init* program. I get it from
[baseimage-docker](https://github.com/phusion/baseimage-docker).
Thanks a lot, you guys.

Procszoo does not require new version Python (but we support python3, too)
and Linux kernel.

## Requirements
---------------

Procszoo only requires Python standard libraries and the following packages

    # on RHEL/CentOS >= 6
    sudo yum -y install autoconf gcc make glibc-headers
    # Debain/Ubuntu
    sudo apt-get -y install autoconf gcc make libc6-dev

On the RHEL/CentOS/Scientific Linux >= 6, because of SELinux
causing some libffi bug, if you try python3, please disable SELinux as follows,

    sudo sed -i -e 's/^SELINUX.*=.*/SELINUX=disabled/g' /etc/selinux/config
    sudo reboot

If you will clone the *procszoo* in your home directory, On
the RHEL/CentOS/Scientific Linux/Fedora, the default mode of your home
directory is 0400, this will cause trouble, hence change it

    chmod go+rx ${HOME}

Building
--------
All you need do are to clone it and do as follows,

    git clone https://github.com/xning/procszoo.git
    cd procszoo && make

Try It
------
Now you can try it in an interactive shell as follows

    cd bin
    # what namsepaces are available?
    ./richard_parker -l
    # what C functions are available?
    ./richard_parker --available-c-functions
    # get an interactive shell
    ./richard_parker

If your Linux kernel doesn't support "user" namespaces, e.g., RHEL6/CentOS6,
RHEL7/CentOS7, you need *super user* privileges

    sudo ./richard_parker

And now, you can check sth that we are in namespaces

* programs get samll pids, e.g., 1, 2, etc., and there is only *lo* device
and it is down

        ps -ef
        ifconfig -a

* open another terminal, we can see that the namespaces entries are different
from our namespaces

        ls -l /proc/self/ns

* if the kernel support and enable "user" namespaces, we are *superuser* now

        id

* if you have trouble to try the above steps, please reference
[Known Issues](#known-issues).


## Getting Your Feet Wet with the *procszoo* module
---------------------------------------------------

First, make sure that the *procszoo* module path in the *sys.path*. You
can add the path as follows

    import sys
    sys.path.append(path_to_procszoo)

then if you want to enable each namespaces that your kernel supports

    from procszoo.c_functions import *
    
    if __name__ == "__main__":
        spawn_namespaces()

If you need run your own program instead of an interactive *shell*, 

    from procszoo.c_functionss import *
    
    if __name__ == "__main__":
        spawn_namespaces(nscmd=path_to_your_program)

## Networks
-----------

Let's add network to the new namespaces.

Because we will mount namespaces entries by the *bind* flag, we need
run *richard_parker* as the super user.

Except the shell that *richard_parker* will open, we need another
interactive shell to make *veth* devices and add them to the new
"net" namespace.

* create a mount point

        mkdir /tmp/ns

* create namespaces

        sudo ./richard_parker --ns_bind_dir=/tmp/ns

* in *richard_parker*, configure the *lo* device

        ip link set lo up

* in a new terminal, remount the */tmp/ns/net* to */var/run/netns/net*
so *ip* command could operate it

        [ -d /var/run/netns ] | sudo mkdir -p  /var/run/netns
        sudo touch /var/run/netns/ns
        sudo mount --bind /tmp/ns/net /var/run/netns/ns

* in the new terminal, create two devices and set one of it to the new
namespace in a new terminal

        sudo ip link add veth0 type veth peer name veth1
        sudo ip link set dev veth1 netns ns

* in the new terminal, configure *veth0* device

        sudo ip link set veth0 up
        sudo ip addr add 192.168.0.10/24 broadcast 192.168.0.255 dev veth0

* in *richard_parker*, configure *veth1*

        ip link set veth1 up
        ip addr add 192.168.0.11/24 broadcast 192.168.0.255 dev veth1

* let's say "hello" from the new terminal

        ping -c 3 192.168.0.11

* let's say "hello" from *richard_parker*

        ping -c 3 192.168.0.10

## Docs
-------

* [namespaces(7)](http://man7.org/linux/man-pages/man7/namespaces.7.html)

* [unshare(2)](http://man7.org/linux/man-pages/man2/unshare.2.html)

* [setns(2)](http://man7.org/linux/man-pages/man2/setns.2.html)

* [ip(8)](http://man7.org/linux/man-pages/man8/ip.8.html)

* [mount(2)](http://man7.org/linux/man-pages/man2/mount.2.html)

* [pivot_root(2)](http://man7.org/linux/man-pages/man2/pivot\_root.2.html)

* [Linux namespaces](https://en.wikipedia.org/wiki/Linux_namespaces)

* [Docker and the PID 1 zombie reaping problem](https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/)

* [Resource management:Linux kernel Namespaces and cgroups](http://www.haifux.org/lectures/299/netLec7.pdf)

* [Containers and Namespaces in the Linux Kernel](https://events.linuxfoundation.org/slides/lfcs2010_kolyshkin.pdf)

## Known Issues
---------------

* os.execv complains "permission deny"

    If running *richard_parker* failed on RHEL/CentOS/Fedora, and get
    following error message like this

    >         os.execv(...)
    >     OSError: [Errno 13] Permission denied

    That's not a bug, please see
    the [comment](https://bugzilla.redhat.com/show_bug.cgi?id=1349789#c7).

* "ip netns" failed on RHRL6/CentOS6 and gave error messages as follows

    >     Object "nets" is unknown, try "ip help".

    We need a more latest iproute package, to do that pls reference
    [here](https://github.com/xning/procszoo/wiki/How-to-build-iproute-and-python-pyroute2-that-supports-net-namespace%3F)

## Exported Functions and Objects
--------------------------------

The *procszoo.utils* exported following functions and objects, and I don't
think that you need learn them all

* objects
    - workbench

* key functions
    - [spawn\_namespaces](https://github.com/xning/procszoo/wiki/The-spawn_namespace-method-workflow)
    - check\_namespaces\_available\_status

* helpful functions
    - atfork
    - sched\_getcpu
    - mount
    - umount
    - umount2
    - unshare
    - setns
    - gethostname
    - sethostname
    - getdomainname
    - setdomainname
    - pivot\_root
    - to\_unicode
    - to\_bytes
    - adjust\_namespaces
    - get_namespace
    - show\_namespaces\_status
    - show\_available\_c\_functions
    - cgroup\_namespace\_available
    - ipc\_namespace\_available
    - net\_namespace\_available
    - mount\_namespace\_available
    - pid\_namespace\_available
    - user\_namespace\_available
    - uts\_namespace\_available
    - unregister\_fork\_handlers

* Exceptions
    - CFunctionBaseException
    - CFunctionNotFound
    - NamespaceGenericException
    - UnknownNamespaceFound
    - UnavailableNamespaceFound
    - NamespaceSettingError

## Test Platforms
----------------
I test the *richard_parker* and these scripts in *tests/* on following
archs

- CentOS 6(x86)
- CentOS 7(x86\_64)
- Fedora 24(x86\_64)
- ubuntu 12.04(armv7l)
- Ubuntu 14.04(x86\_64)
- Ubuntu 16.04(x86\_64)

