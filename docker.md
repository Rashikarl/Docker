# Docker
Docker.md
Skip to content
FreddieOliveira/docker.md
Last active 18 hours ago • Report abuse
Unstar this gist
Code
Revisions
64
Stars
838
Forks
112
This tutorial shows how to run docker natively on Android, without VMs and chroot.
docker.md
Docker on Android 🐋📱
Edit 🎉
All packages, except for Tini have been added to termux-root. To install them, simply pkg install root-repo && pkg install docker. This will install the whole docker suite, left only Tini to be compiled manually.

Summary
Intro
Building
Rooting
Kernel
General compiling instructions
Modifications
Patching
Docker
dockercli
dockerd
tini
libnetwork
containerd
runc
Running
Caveats
Internet access
Shared volumes
GUI
X11 Forwarding
VNC server within the container
Steam (work in progress)
Attachments
Kernel patches
docker-cli patches
dockerd patches
containerd patches
Aknowledgements
Final notes
1. Intro
This tutorial presents a step by step guide on how to run docker containers directly on Android. By directly I mean there's no VM involved nor chrooting inside a GNU/Linux rootfs. This is docker purely in Android. Yes, it is possible.

Bear in mind that you'll have to root your phone, mess with and compile your phone's kernel and docker suite. So, be prepared to get your hands dirty.

2. Building
2.1. Rooting
This step is pretty device specific, so there's no way to write a generic tutorial here. You'll need to google for instructions for your device and follow them.

Just be aware that you may lose your phone's warrant and all your data will be erased after unlocking the bootloader, so make a backup of your important stuff.

2.2. Kernel
2.2.1. General compiling instructions
Compiling the phone's kernel is also device specific, but some major tips may help you out.

First, google about instructions for your phone. Start by compiling the kernel without any modification. Flash it and hope for the best. If everything went well, then you can proceed to the modifications.

Note that flashing the kernel won't erase any data in your phone. The worst that can happen is you get stuck in a boot loop. In this case, you can flash a kernel that's known to be working or just flash a working ROM, since it contains a kernel with it. None of these operations erase any data in your phone.

2.2.2. Modifications
Now that you (hopefully) are able to compile the kernel, let's talk about what matters. Docker needs a lot of features that are disabled by default in Android's kernel.

To check the necessary features list, first install the Termux app in your phone. This is the terminal emulator that we're going to use throughout this guide. It has a package manager with many tools compiled for Android.

Next, open Termux and download a script to check your kernel:

$ pkg install wget
$ wget https://raw.githubusercontent.com/moby/moby/master/contrib/check-config.sh
$ chmod +x check-config.sh
$ sed -i '1s_.*_#!/data/data/com.termux/files/usr/bin/bash_' check-config.sh
$ sudo ./check-config.sh
Now, in your computer, open the kernel's configuration menu. This menu is a modified version of dialog, a ncurses window menu, in which you can enable and disable the kernel features. To look for some item in particular, you can press the / key and type the item name and hit Enter. This will show the description and location of the item.

For now, we want to enable the Generally Necessary items, the Network Drivers items and some Optional Features. For the Storage Drivers we'll be using the overlay.

2.2.3. Patching
Before compiling the kernel there are two files that need to be patched.

kernel/Makefile
The first one is the kernel/Makefile. Although not strictly necessary to modify this file, it will help by making it possible to check if your kernel has all the necessary features docker needs.

If you do not apply this patch, the output of the check-config.sh script used above won't be reliable after recompiling the kernel.

Check the patch at the attachments section and modify your Makefile accordingly.

net/netfilter/xt_qtaguid.c
This second file needs to be patched because of a bug introduced by Google. After you run any container, a seg fault will be generated due to a null pointer dereference and your phone will freeze and reboot. If you work at Google or know someone who does, warn him/her about it.

Check the patch at the attachments section and modify your xt_qtaguid.c accordingly.

Now that everything is setup, compile and flash the kernel. If you applied the Makefile patch, you'll see this warning everytime your phone boots:

IMG_20210110_203818

Don't worry though, this is a harmless warning remembering you that you're using a modified kernel.

2.3. Docker
See Edit.

Once you have a supported kernel, it's time to compile the docker suite. It's a suite because it's not just one program, but rather a set of different programs that we'll need to compile separately. So hands on.

Firts, let's install the packages we're gonna use to build docker in Termux:

$ pkg install go make cmake ndk-multilib tsu
Now we're ready to start compiling things. Create a work directory where the packages will be downloaded and built:

$ mkdir $TMPDIR/docker-build
$ cd $TMPDIR/docker-build
Download all the patches files into there and let's begin. All commands for the differents packages that'll be compiled next is meant to be executed inside this folder.

2.3.1. dockercli
See Edit.

This is the docker client, which will talk to the docker daemon. This package will compile a binary named docker and all docker man pages. To build and install it:

$ cd $TMPDIR/docker-build
$ wget https://github.com/docker/cli/archive/v20.10.2.tar.gz -O cli-20.10.2.tar.gz
$ tar xf cli-20.10.2.tar.gz
$ mkdir -p src/github.com/docker
$ mv cli-20.10.2 src/github.com/docker/cli
$ export GOPATH=$(pwd)
$ export VERSION=v20.10.2-ce
$ export DISABLE_WARN_OUTSIDE_CONTAINER=1
$ cd src/github.com/docker/cli
$ xargs sed -i 's_/var/\(run/docker\.sock\)_/data/docker/\1_g' < <(grep -R /var/run/docker\.sock | cut -d':' -f1 | sort | uniq)
$ patch vendor/github.com/containerd/containerd/platforms/database.go ../../../../database.go.patch.txt
$ patch scripts/docs/generate-man.sh ../../../../generate-man.sh.patch.txt
$ patch man/md2man-all.sh ../../../../md2man-all.sh.patch.txt
$ patch cli/config/config.go ../../../../config.go.patch.txt
$ make dynbinary
$ make manpages
$ install -Dm 0700 build/docker-android-* $PREFIX/bin/docker
$ install -Dm 600 -t $PREFIX/share/man/man1 man/man1/*
$ install -Dm 600 -t $PREFIX/share/man/man5 man/man5/*
$ install -Dm 600 -t $PREFIX/share/man/man8 man/man8/*
2.3.2. dockerd
See Edit.

The docker daemon is the most problematic binary that's gonna be compiled. It needs so many patches that's easier to modify the code in a batch with sed. Despite the need of modifying a lot of files, the modifications by themselfs are rather simple:

Substitute every occurrence of runtime.GOOS by the string "linux";
Remove unneeded imports of the runtime lib.
By doing that, we are in essence spoofing our operating system as a Linux one: everytime the code would do the runtime.GOOS == "linux" comparison (which would become "android" == "linux", and thus false) it will now do "linux" == "linux" and thus true.

To make the substitution across every file, we'll run a sed command. After that, some files will now give the extremely annoying unturnable-off go lang "feature" imported and not used error, because the only function these files were using from the runtime package was the runtime.GOOS. So, to fix it we'll use an horrible but simple solution: we'll keep trying to compile the code and at each failed attempt we'll fix the reported files till we get it to compile successfully.

$ cd $TMPDIR/docker-build
$ wget https://github.com/moby/moby/archive/v20.10.2.tar.gz -O moby-20.10.2.tar.gz
$ tar xf moby-20.10.2.tar.gz
$ cd moby-20.10.2
$ export DOCKER_GITCOMMIT=8891c58a43
$ export DOCKER_BUILDTAGS='exclude_graphdriver_btrfs exclude_graphdriver_devicemapper exclude_graphdriver_quota selinux exclude_graphdriver_aufs'
$ patch cmd/dockerd/daemon.go ../daemon.go.patch
$ xargs sed -i "s_\(/etc/docker\)_$PREFIX\1_g" < <(grep -R /etc/docker | cut -d':' -f1 | sort | uniq)
$ xargs sed -i 's_\(/run/docker/plugins\)_/data/docker\1_g' < <(grep -R '/run/docker/plugins' | cut -d':' -f1 | sort | uniq)
$ xargs sed -i 's/[a-zA-Z0-9]*\.GOOS/"linux"/g' < <(grep -R '[a-zA-Z0-9]*\.GOOS' | cut -d':' -f1 | sort | uniq)
$ (while ! IFS='' files=$(AUTO_GOPATH=1 PREFIX='' hack/make.sh dynbinary 2>&1 1>/dev/null); do if ! xargs sed -i 's/\("runtime"\)/_ \1/' < <(echo $files | grep runtime | cut -d':' -f1 | cut -c38-); then echo $files; exit 1; fi; done)
$ install -Dm 0700 bundles/dynbinary-daemon/dockerd $PREFIX/bin/dockerd-dev
A binary called dockerd-dev was compiled and installed, but in order to it run correctly, the cgroups need to be mounted. Since Android mounts the cgroups in a non standard location we need to fix this. To do so, a script named dockerd will be created that will mount crgoups in the correct path if needed and call dockerd-dev next.

$ cat << "EOF" > $PREFIX/bin/dockerd
#!/data/data/com.termux/files/usr/bin/bash

export PATH="${PATH}:/system/xbin:/system/bin"
opts='rw,nosuid,nodev,noexec,relatime'
cgroups='blkio cpu cpuacct cpuset devices freezer memory pids schedtune'

# try to mount cgroup root dir and exit in case of failure
if ! mountpoint -q /sys/fs/cgroup 2>/dev/null; then
  mkdir -p /sys/fs/cgroup
  mount -t tmpfs -o "${opts}" cgroup_root /sys/fs/cgroup || exit
fi

# try to mount cgroup2
if ! mountpoint -q /sys/fs/cgroup/cg2_bpf 2>/dev/null; then
  mkdir -p /sys/fs/cgroup/cg2_bpf
  mount -t cgroup2 -o "${opts}" cgroup2_root /sys/fs/cgroup/cg2_bpf
fi

# try to mount differents cgroups
for cg in ${cgroups}; do
  if ! mountpoint -q "/sys/fs/cgroup/${cg}" 2>/dev/null; then
    mkdir -p "/sys/fs/cgroup/${cg}"
    mount -t cgroup -o "${opts},${cg}" "${cg}" "/sys/fs/cgroup/${cg}" \
    || rmdir "/sys/fs/cgroup/${cg}"
  fi
done

# start the docker daemon
$PREFIX/bin/dockerd-dev $@
EOF
Make the script executable:

$ chmod +x $PREFIX/bin/dockerd
And finally configure some dockerd options:

$ mkdir -p $PREFIX/etc/docker
$ cat << "EOF" > $PREFIX/etc/docker/daemon.json
{
    "data-root": "/data/docker/lib/docker",
    "exec-root": "/data/docker/run/docker",
    "pidfile": "/data/docker/run/docker.pid",
    "hosts": [
        "unix:///data/docker/run/docker.sock"
    ],
    "storage-driver": "overlay2"
}
EOF
Warning: dockerd will store all its files, like containers, images, volumes, etc inside the /data/docker folder, which means you'll lose everything if you format the phone (flash a ROM). This folder was chosen instead of storing things inside Termux installation folder, because dockerd fails when setting up the overlay storage driver there. It seems Android mounts the /data/data folder with some options that prevent overlayfs to work, or the filesystem doesn't support it.

2.3.3. tini
tini is an optional dependency of dockerd in case you want the init process to be the first process of the container being ran (for this use the --init flag when creating a container). Having init as the parent of all other proccess ensures that a proper clean up inside the container is made regarding zombie processes. For a detailed explanation on its benefits and when to use it, check here: krallin/tini#8

$ cd $TMPDIR/docker-build
$ wget https://github.com/krallin/tini/archive/v0.19.0.tar.gz
$ tar xf v0.19.0.tar.gz
$ cd tini-0.19.0
$ mkdir build
$ cd build
$ cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$PREFIX ..
$ make -j8
$ make install
$ ln -s $PREFIX/bin/tini-static $PREFIX/bin/docker-init
2.3.4. libnetwork
See Edit.

Another dockerd dependency needed when using the -p flag while creating a container:

$ cd $TMPDIR/docker-build
$ wget https://github.com/moby/libnetwork/archive/448016ef11309bd67541dcf4d72f1f5b7de94862.tar.gz
$ tar xf 448016ef11309bd67541dcf4d72f1f5b7de94862.tar.gz
$ mkdir -p src/github.com/docker
$ mv libnetwork-448016ef11309bd67541dcf4d72f1f5b7de94862 src/github.com/docker/libnetwork
$ export GOPATH="$(pwd)"
$ cd src/github.com/docker/libnetwork
$ go build -o docker-proxy github.com/docker/libnetwork/cmd/proxy
$ strip docker-proxy
$ install -Dm 0700 docker-proxy $PREFIX/bin/docker-proxy
2.3.5. containerd
See Edit.

This is a dockerd dependency. Some patches are needed to fix path locations, build the manuals correctly and compile extra binaries used by dockerd that are not build by default by the Makefile:

$ cd $TMPDIR/docker-build
$ wget https://github.com/containerd/containerd/archive/v1.4.3.tar.gz
$ tar xf v1.4.3.tar.gz
$ mkdir -p src/github.com/containerd
$ mv containerd-1.4.3 src/github.com/containerd/containerd
$ export GOPATH=$(pwd)
$ cd src/github.com/containerd/containerd
$ xargs sed -i "s_\(/etc/containerd\)_$PREFIX\1_g" < <(grep -R /etc/containerd | cut -d':' -f1 | sort | uniq)
$ patch runtime/v1/linux/bundle.go ../../../../bundle.go.patch.txt
$ patch runtime/v2/shim/util_unix.go ../../../../util_unix.go.patch.txt
$ patch Makefile ../../../../Makefile.patch
$ patch platforms/database.go ../../../../database.go.patch.txt
$ patch vendor/github.com/cpuguy83/go-md2man/v2/md2man.go ../../../../md2man.go.patch.txt
$ BUILDTAGS=no_btrfs make -j8
$ make -j8 man
$ DESTDIR=$PREFIX make install
$ DESTDIR=$PREFIX/share make install-man
Lastly, some configurations:

$ mkdir -p $PREFIX/etc/containerd
$ cat << "EOF" > $PREFIX/etc/containerd/config.toml
root = "/data/docker/var/lib/containerd"
state = "/data/docker/run/containerd"
imports = ["$PREFIX/etc/containerd/runtime_*.toml", "./debug.toml"]

[grpc]
  address = "/data/docker/run/containerd/containerd.sock"

[debug]
  address = "/data/docker/run/containerd/debug.sock"

[plugins]
  [plugins.opt]
    path = "/data/docker/opt"
  [plugins.cri.cni]
    bin_dir = "/data/docker/opt/cni/bin"
    conf_dir = "/data/docker/etc/cni/net.d"
EOF
Note: unfortunately containerd files also can't be stored inside Termux installation folder, failing with an error when creating the socket it uses.

2.3.6. runc
See Edit.

runc is a dependency of containerd. Conveniently for us, it's already provided by Termux's repository. Install it by simply:

$ pkg install root-repo
$ pkg install runc
3. Running
Now comes the truth time. To run the containers, first we need to start the daemon manually. To do so, it's advisable to install a terminal multiplexer so you can run the daemon in one pane and the container in others panes:

$ pkg install tmux
In one pane start dockerd:

$ sudo dockerd --iptables=false
And in others panes you can run the containers:

$ sudo docker run hello-world
Note: Teaching how to use tmux is out of the scope of this guide, you can find good tutorials on YouTube. If you don't wanna use a terminal multiplexer, you can run dockerd in the background instead, with sudo dockerd &>/dev/null &.

3.1. Caveats
3.1.1. Internet access
The two network drivers tested so far are bridge and host. Here's how to get each of them working.

bridge
This is the default netwok driver. If you don't specify a driver, this is the type of network you are creating. Bridge networks isolate the container network by editing the iptables rules and creating a network interface called Docker0 that serves as a bridge. All containers created with the bridge driver will use this interface. This is analogous to creating a VLAN and running the containers inside it.

But, there's a catch in Android: iptables rules policy is different here than on a conventional GNU/Linux system (more info here). For the bridge driver to work, you'll have to manually edit the iptable by running;

$ sudo ip route add default via 192.168.1.1 dev wlan0
$ sudo ip rule add from all lookup main pref 30000
Note: change 192.168.1.1 according to your gateway IP.

Unfortunately, this means that changing networks will require you to re-configure the rules again.

host
Using the host driver, means to remove network isolation between the container and the Docker host, and use the host’s networking directly. This way, the container will use the same network interface as your device (e.g. wlan0) and thus will share the same IP address.

To use this driver give the --net=host --dns=8.8.8.8 flags when running a container.

3.1.2. Shared volumes
An easy way to share folders and files between containers and the host is to use a shared volume. For example, using the -v ~/Documents/docker-share:/root/docker-share flag when running a container, will make the ~/Documents/docker-share folder from the host to be accessible inside the container /root/docker-share folder.

But, when talking about Android, things seems to never be as easy and straightforward as expected. Due to Android file system encryption, if you ls the /root/docker-share folder inside the container you might see a bunch of random letters, numbers and symbols instead of the folders and files names:

# ls /root/docker-share
+2xKy7JIRrcGrCf+o6KSeB  T6BJkyIa5OedXNrSyRKLbB  cwoDh,Nzt1l,5BsKA4hH8D
2aHRCQEyK8yYiiK9PEI9SA  Ue39lJVm4kIxGrS1bV07zB  lEpWZhTY9dNqJxCu+GqBuA
5ZRDLfHMwyik6RMe,f0WPA  X+yGLxXSgwxbCsFGRXuczC  y4ZWVvVBBjcxSWlJ9conED
GljgSZK5gFr7D4Fk7BHNeB  X1ATNoqhp,,ZsKjFXqKFiA
I3N5j0R4zmaQPKCWwKBlxD  Yzi+KmovJmIYFOCHtDCXkB
and if you try to read or create a file inside the volume you might get the Required key not available error.

No definitive solution was discovered so far, but a workaround is to cat the files from within the host to give the container temporary access to them. You can cat an individual file by:

$ sudo cat ~/Documents/docker-share/file.pdf >/dev/null
or all of them by:

$ sudo find ~/Documents/docker-share -exec cat {} >/dev/null \;
3.2. GUI
Yes, it's possible to run GUI programs inside a container! There's basically two ways of accomplishing it in a simple manner:

3.2.1. X11 Forwarding
Description
This method has the advantage of not making necessary the installation nor configuration of any additional programs inside the container; all you'll have to do is to setup the X inside termux and share its sockets with the container.

This is advisable to be used when you intend to run various containers with GUI, since you'll only have to install and setup a VNC once in the host, instead of doing it for each container. This will save storage space and time.

Steps
The first step is to enable the X11 repository in termux, this will allow installation of graphical interface related programs, like the VNC server we'll be using.

$ pkg install x11-repo
Then install a VNC server in termux:

$ pkg install tigervnc
Note: These installations steps need to be executed only once.

Now, just run it:

$ vncserver -noxstartup -localhost
Note: It's advisable to pass the -geometry HEIGHTxWEIGHT flag substituting HEIGHT and WEIGHT by your phone's screen resolution or some multiple of it.

Note: The very first time you run it, you'll be prompted to setup a password. Note that passwords are not visible when you are typing them and it's maximal length is 8 characters. If you don't wanna use a passwd, use the -SecurityTypes none flag.

If everything is okay, you will see this message:

New 'localhost:1 ()' desktop is localhost:1
It means that X (vnc) server is available on display 'localhost:1'. Finally, export the DISPLAY environment variable according to that value:

$ export DISPLAY=:1
Now that the VNC server is configured and running in the host, start the container sharing the X related files as volumes:

$ sudo docker run -ti \
    --net="host" \
    --dns="8.8.8.8" \
    -e DISPLAY=$DISPLAY \
    -v $TMPDIR/.X11-unix:/tmp/.X11-unix \
    -v $HOME/.Xauthority:/root/.Xauthority \
    ubuntu
Note: If by any reason you forget to export the DISPLAY before starting the container, you can still export it from inside it.

You'll now be able to launch GUI programs from inside the container, e.g.:

# echo 'APT::Sandbox::User "root";' > /etc/apt/apt.conf
# apt update
# apt install x11-apps
# xeyes
To check the GUI, you'll need to install a VNC client app in your Android phone, like VNC Viewer (developed by RealVNC Limited). Unfortunately it's not open source, but it's a good and intuitive VNC client for Android.

Note: There's also an open source alternative developed by @pelya called XServer XSDL, which will not be covered by this guide (for now).

After installing the VNC Viewer app, open it and setup a new connection using 127.0.0.1 (or localhost) as the IP, 5901 as the port (the port is calculated as 5900 + {display number}) and when/if prompted, type the password choosen when running vnctiger for the first time.

3.2.2. VNC server within the container
Description
This method is very similar to the previous, with the difference that the X server will be installed inside the container instead of in the termux host.

The advantages are:

you aren't changing your host system by installing softwares on it (like the VNC server);
security, since you won't be sharing your host's X (this is only relevant when you are not the one running the container).
The main disadvantage is that you'll need to install and config the VNC server for each container you'd run a GUI program, thus making these containers big and time consuming to setup.

Steps
First, start a container:

$ sudo docker run -ti \
    --net="host" \
    --dns="8.8.8.8" \
    ubuntu
Now, a VNC server needs to be installed and configured inside the container. You can choose between TigerVNC or x11vnc:

TigerVNC
The same VNC server used above in termux. To install it:

# echo 'APT::Sandbox::User "root";' > /etc/apt/apt.conf
# apt update
# apt install tigervnc-standalone-server
Next, start it with:

# vncserver -noxstartup -localhost -SecurityTypes none
Here we disabled password (-SecurityTypes none) because using it causes things to crash as described in this issue report TigerVNC/tigervnc#800

If everything is okay, you will see this message:

New 'localhost:1 (root)' desktop at :1 on machine localhost
Export the DISPLAY environment variable according to that value:

# export DISPLAY=:1
From now on, you can already run GUI programs and access them using the VNC Viewer client as already described in the end of X11 Forwarding steps.

x11vnc
Install the x11vnc and the virtual fake X (since x11vnc can't emulate a X11 by itself):

# echo 'APT::Sandbox::User "root";' > /etc/apt/apt.conf
# apt update
# apt install x11vnc xvfb
Now, start it:

# x11vnc -nopw -forever -noshm -create
If everything is okay, you will see this message:

The VNC desktop is:      localhost:0
PORT=5900
This will open a xterm terminal which can be acessed by the VNC Viewer client as already described in the end of X11 Forwarding steps. From that terminal you can open the desired GUI program.

3.3. Steam (work in progress)
I'm not talking about running the useless steam app for Android, but about running the Desktop version and play the games inside a docker container. Yes, you read it right, it's possible to play your Steam games on Android!

(ACTUALLY NOT YET, BECAUSE I DIDN'T MANAGE TO GET OPENGL TO WORK, THAT'S WHY THIS IS A WORK IN PROGRESS. TO CONTRIBUTE OR STAY UP TO DATE ABOUT THE PROGRESS CHECK ptitSeb/box86#249)

To do so, we'll use an awesome x86 emulator for ARM developed by @ptitSeb called box86.

But first, you need to enable System V IPC under General Setup in the kernel config and recompile it again. That's because the steam binary needs some semaphore functions and will crash in case it can't use them.

Next, we hit a problem: box86 can only be compiled by a 32 bit toolchain. But, in fact, this can be easily circumvented by using a 32 bit container:

$ sudo docker run -ti \
    --net="host" \
    --dns="8.8.8.8" \
    -e DISPLAY=$DISPLAY \
    -w /root \
    -v $TMPDIR/.X11-unix:/tmp/.X11-unix \
    -v $HOME/.Xauthority:/root/.Xauthority \
    --platform=linux/arm \
    arm32v7/ubuntu
Note: if your system is 32 bit already (run uname -m to check), you don't need to specify the --platform=linux/arm flag and can simply use ubuntu instead of arm32v7/ubuntu.

Now that we are inside the container, let's install the tools we're gonna use, as well as the steam .deb installer:

# echo 'APT::Sandbox::User root;' >> /etc/apt/apt.conf
# apt update
# apt install wget binutils xterm libvdpau1 libappindicator1 libnm0 libdbusmenu-gtk4
Install steam:

# wget https://steamcdn-a.akamaihd.net/client/installer/steam.deb
# ar x steam.deb
# mkdir steam
# tar xf data.tar.xz -C steam
# find steam -type d -exec sh -c 'mkdir -p /${0#*/}' {} \;
# find steam \! -type d -exec sh -c 'mv $0 /${0#*/}' {} \;
# patch /usr/lib/steam/bin_steam.sh bin_steam.sh.patch
# rm -rf steam* *.tar* bin_steam.sh.patch
# steam
Steam will fail with a bunch of errors, but that's expected. The important thing is that it installed the necessary files under ~/.local/share/Steam, one of them being the steam binary. Finish the installation by adding it to the path:

# ln -sf /root/.local/share/Steam/ubuntu12_32/steam /usr/bin/steam
Now, we need to install the i386 version of some libs required by steam. For this, we're going to download them directly from Ubuntu packages. That's because if we instead simply apt install them we would be getting the arm32 version.

4. Attachments
4.1. kernel patches
kernel/Makefile
diff --git a/kernel/Makefile b/kernel/Makefile
index d5c1115..2dea801 100644
--- a/kernel/Makefile
+++ b/kernel/Makefile
@@ -121,7 +121,7 @@ $(obj)/configs.o: $(obj)/config_data.h
# config_data.h contains the same information as ikconfig.h but gzipped.
# Info from config_data can be extracted from /proc/config*
targets += config_data.gz
-$(obj)/config_data.gz: arch/arm64/configs/lavender_stock-defconfig FORCE
+$(obj)/config_data.gz: $(KCONFIG_CONFIG) FORCE
    $(call if_changed,gzip)

    filechk_ikconfiggz = (echo "static const char kernel_config_data[] __used = MAGIC_START"; cat $< | scripts/basic/bin2c; echo "MAGIC_END;")
net/netfilter/xt_qtaguid.c
--- orig/net/netfilter/xt_qtaguid.c     2020-05-12 12:13:14.000000000 +0300
+++ my/net/netfilter/xt_qtaguid.c       2019-09-15 23:56:45.000000000 +0300
@@ -737,7 +737,7 @@
{
        struct proc_iface_stat_fmt_info *p = m->private;
        struct iface_stat *iface_entry;
-       struct rtnl_link_stats64 dev_stats, *stats;
+       struct rtnl_link_stats64 *stats;
        struct rtnl_link_stats64 no_dev_stats = {0};  
@@ -745,13 +745,8 @@
        current->pid, current->tgid, from_kuid(&init_user_ns, current_fsuid()));
        iface_entry = list_entry(v, struct iface_stat, list);
+       stats = &no_dev_stats; 
-       if (iface_entry->active) {
-               stats = dev_get_stats(iface_entry->net_dev,
-                                     &dev_stats);
-       } else {
-               stats = &no_dev_stats;
-       }
        /*
         * If the meaning of the data changes, then update the fmtX
         * string.
4.2. docker-cli patches
vendor/github.com/containerd/containerd/platforms/database.go
scripts/docs/generate-man.sh
man/md2man-all.sh
cli/config/config.go
4.3. dockerd patches
cmd/dockerd/daemon.go
4.4. containerd patches
runtime/v1/linux/bundle.go
runtime/v2/shim/util_unix.go
Makefile
platforms/database.go
vendor/github.com/cpuguy83/go-md2man/v2/md2man.go
5. Aknowledgements
I'd like to thank the Termux Dev team for this wonderful app and @xeffyr for discovering about the bug in net/netfilter/xt_qtaguid.c and sharing the patch, as well as all the conversation we had here that led to docker finally working.

Also @yjwong, for figuring out how to use the bridge network driver.

6. Final notes
If you are a docker developer reading this, please consider adding an official support for Android. Look above the possibilities it opens for a smartphone. If you are not a docker developer, consider supporting this by showing interest here. If we annoy the devs enough, this may become official (of they may simply unsubscribe from the thread and let it rot in the Issues section ¯\_(ツ)_/¯ ).

Load earlier comments...
WLFJ commented on Feb 19
i am getting the error

 ~
➜ sudo dockerd --iptables=false
INFO[2023-02-18T23:25:49.776669495Z] Starting up
WARN[2023-02-18T23:25:49.788914443Z] could not change group /data/docker/run/docker.sock to docker: group docker not found
INFO[2023-02-18T23:25:49.796665328Z] libcontainerd: started new containerd process  pid=20704
INFO[2023-02-18T23:25:49.797694755Z] [core] [Channel #1] Channel created           module=grpc
INFO[2023-02-18T23:25:49.797851422Z] [core] [Channel #1] original dial target is: "unix:///data/docker/run/docker/containerd/containerd.sock"  module=grpc
INFO[2023-02-18T23:25:49.798039599Z] [core] [Channel #1] parsed dial target is: {Scheme:unix Authority: Endpoint:data/docker/run/docker/containerd/containerd.sock URL:{Scheme:unix Opaque: User: Host: Path:/data/docker/run/docker/containerd/containerd.sock RawPath: OmitHost:false ForceQuery:false RawQuery: Fragment: RawFragment:}}  module=grpc
INFO[2023-02-18T23:25:49.798111526Z] [core] [Channel #1] Channel authority set to "localhost"  module=grpc
INFO[2023-02-18T23:25:49.804918974Z] [core] [Channel #1] Resolver state updated: {
  "Addresses": [
    {
      "Addr": "/data/docker/run/docker/containerd/containerd.sock",
      "ServerName": "",
      "Attributes": {},
      "BalancerAttributes": null,
      "Type": 0,
      "Metadata": null
    }
  ],
  "ServiceConfig": null,
  "Attributes": null
} (resolver returned new addresses)  module=grpc
INFO[2023-02-18T23:25:49.806475588Z] [core] [Channel #1] Channel switches to new LB policy "pick_first"  module=grpc
INFO[2023-02-18T23:25:49.809323036Z] [core] [Channel #1 SubChannel #2] Subchannel created  module=grpc
INFO[2023-02-18T23:25:49.809856734Z] [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to CONNECTING  module=grpc
INFO[2023-02-18T23:25:49.810129026Z] [core] [Channel #1 SubChannel #2] Subchannel picks a new address "/data/docker/run/docker/containerd/containerd.sock" to connect  module=grpc
INFO[2023-02-18T23:25:49.810327047Z] [core] [Channel #1] Channel Connectivity change to CONNECTING  module=grpc
containerd: failed to load TOML from /data/docker/run/docker/containerd/containerd.toml: invalid plugin key URI "opt" expect io.containerd.x.vx
ERRO[2023-02-18T23:25:50.120531265Z] containerd did not exit successfully          error="exit status 1" module=libcontainerd
WARN[2023-02-18T23:25:50.811922463Z] [core] [Channel #1 SubChannel #2] grpc: addrConn.createTransport failed to connect to {
  "Addr": "/data/docker/run/docker/containerd/containerd.sock",
  "ServerName": "localhost",
  "Attributes": {},
  "BalancerAttributes": null,
  "Type": 0,
  "Metadata": null
}. Err: connection error: desc = "transport: error while dialing: dial unix:///data/docker/run/docker/containerd/containerd.sock: timeout"  module=grpc
INFO[2023-02-18T23:25:50.812241005Z] [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to TRANSIENT_FAILURE  module=grpc
INFO[2023-02-18T23:25:50.812393244Z] [core] [Channel #1] Channel Connectivity change to TRANSIENT_FAILURE  module=grpc
INFO[2023-02-18T23:25:51.812556265Z] [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to IDLE  module=grpc
INFO[2023-02-18T23:25:51.812940015Z] [core] [Channel #1] Channel Connectivity change to IDLE  module=grpc
INFO[2023-02-18T23:25:51.813146577Z] [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to CONNECTING  module=grpc
INFO[2023-02-18T23:25:51.813275171Z] [core] [Channel #1 SubChannel #2] Subchannel picks a new address "/data/docker/run/docker/containerd/containerd.sock" to connect  module=grpc
INFO[2023-02-18T23:25:51.813828765Z] [core] [Channel #1] Channel Connectivity change to CONNECTING  module=grpc
WARN[2023-02-18T23:25:53.526555795Z] [core] [Channel #1 SubChannel #2] grpc: addrConn.createTransport failed to connect to {
  "Addr": "/data/docker/run/docker/containerd/containerd.sock",
  "ServerName": "localhost",
  "Attributes": {},
  "BalancerAttributes": null,
  "Type": 0,
  "Metadata": null
}. Err: connection error: desc = "transport: error while dialing: dial unix:///data/docker/run/docker/containerd/containerd.sock: timeout"  module=grpc
INFO[2023-02-18T23:25:53.526793556Z] [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to TRANSIENT_FAILURE  module=grpc
INFO[2023-02-18T23:25:53.526900483Z] [core] [Channel #1] Channel Connectivity change to TRANSIENT_FAILURE  module=grpc
INFO[2023-02-18T23:25:55.239899180Z] [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to IDLE  module=grpc
INFO[2023-02-18T23:25:55.240128972Z] [core] [Channel #1] Channel Connectivity change to IDLE  module=grpc
INFO[2023-02-18T23:25:55.240250951Z] [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to CONNECTING  module=grpc
INFO[2023-02-18T23:25:55.240342930Z] [core] [Channel #1 SubChannel #2] Subchannel picks a new address "/data/docker/run/docker/containerd/containerd.sock" to connect  module=grpc
INFO[2023-02-18T23:25:55.240636472Z] [core] [Channel #1] Channel Connectivity change to CONNECTING  module=grpc
WARN[2023-02-18T23:25:57.646474388Z] [core] [Channel #1 SubChannel #2] grpc: addrConn.createTransport failed to connect to {
  "Addr": "/data/docker/run/docker/containerd/containerd.sock",
  "ServerName": "localhost",
  "Attributes": {},
  "BalancerAttributes": null,
  "Type": 0,
  "Metadata": null
}. Err: connection error: desc = "transport: error while dialing: dial unix:///data/docker/run/docker/containerd/containerd.sock: timeout"  module=grpc
INFO[2023-02-18T23:25:57.646684179Z] [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to TRANSIENT_FAILURE  module=grpc
INFO[2023-02-18T23:25:57.646850117Z] [core] [Channel #1] Channel Connectivity change to TRANSIENT_FAILURE  module=grpc
INFO[2023-02-18T23:26:00.052411522Z] [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to IDLE  module=grpc
INFO[2023-02-18T23:26:00.052608241Z] [core] [Channel #1] Channel Connectivity change to IDLE  module=grpc
INFO[2023-02-18T23:26:00.052732616Z] [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to CONNECTING  module=grpc
INFO[2023-02-18T23:26:00.052829595Z] [core] [Channel #1 SubChannel #2] Subchannel picks a new address "/data/docker/run/docker/containerd/containerd.sock" to connect  module=grpc
INFO[2023-02-18T23:26:00.053065585Z] [core] [Channel #1] Channel Connectivity change to CONNECTING  module=grpc
WARN[2023-02-18T23:26:02.936575167Z] [core] [Channel #1 SubChannel #2] grpc: addrConn.createTransport failed to connect to {
  "Addr": "/data/docker/run/docker/containerd/containerd.sock",
  "ServerName": "localhost",
  "Attributes": {},
  "BalancerAttributes": null,
  "Type": 0,
  "Metadata": null
}. Err: connection error: desc = "transport: error while dialing: dial unix:///data/docker/run/docker/containerd/containerd.sock: timeout"  module=grpc
INFO[2023-02-18T23:26:02.936886677Z] [core] [Channel #1 SubChannel #2] Subchannel Connectivity change to TRANSIENT_FAILURE  module=grpc
INFO[2023-02-18T23:26:02.937045115Z] [core] [Channel #1] Channel Connectivity change to TRANSIENT_FAILURE  module=grpc
failed to start containerd: timeout waiting for containerd to start
~ took 16s
❯
currently the docker version in termux-packages has some error. instead, you can try to clone https://github.com/shaunmulligan/termux-packages/tree/revert-docker-bump and build it manually.

whyakari commented on Feb 19
@WLFJ thanks, bro. as I understand already from compiling termux packages, it won't be difficult. thank you again.

benayat commented on Feb 21
@WLFJ I tried building from your fork in https://github.com/shaunmulligan/termux-packages/tree/revert-docker-bump, build is successful but when I run "docker run hello-world", I'm getting this message:
docker: Cannot connect to the Docker daemon at unix:///data/docker/run/docker.sock. Is the docker daemon running?.
what could be the problem?

whyakari commented on Feb 21
@benayat bro, did you compile the packages and installed they? did you start the dockerd daemon?

try deamon dockerd:

sudo dockerd --iptables=false

open new session:

sudo docker run hello-world

remembering that you need the "sudo" command to run it and obviously you need to be root.

benayat commented on Feb 21 via email  • 
all I did was - rooted my device, cloned WLFJs termux-packages fork, and
compiled (on the device) the "docker" package. I used this manual(on device
usage): https://wiki.termux.com/wiki/Building_packages.
after that, I run "sudo dockerd --iptables=false", and I got an error
similar to yours -
"}. Err: connection error: desc = "transport: error while dialing: dial
unix:///data/docker/run/docker/containerd/containerd.sock: timeout" module=
grpc
"
@AkariOficial did it work for you?
whyakari commented on Feb 21
@benayat yeah, worked for me. i have compiled in my repository (aarch64). I literally spent two days trying to find the cause or solution. found out when @WLFJ said that version 23.0.1 was broken. i had to compile for aarch64 (my architecture) and using of the version docker 'v20.10.23-ce'

mcstarkteam commented on Feb 27 via email 
dockerd&nbsp;--iptables=false
…
murilopereirame commented on Feb 27
For those who wants to use docker-compose, run the following commands:

sudo bash
export  DOCKER_HOST=unix:///data/docker/run/docker.sock
docker-compose up
a9udn9u commented on Mar 1
Hey @FreddieOliveira

I was able to get the docker engine running following your tutorial, thank you! What's the docker group equivalent on Android or do we have to run docker commands as root?

Mazoch commented on Mar 1
I managed to get the portainer working and that's it. Nothing else wants to work. There are constantly errors with permissions.

binhvo7794 commented on Mar 14
How to fix?
Screenshot_2023-03-14-15-28-47-64_84d3000e3f4017145260f7618db1d683
Screenshot_2023-03-14-15-28-33-03_84d3000e3f4017145260f7618db1d683

murilopereirame commented on Mar 14 • 
How to fix? Screenshot_2023-03-14-15-28-47-64_84d3000e3f4017145260f7618db1d683 Screenshot_2023-03-14-15-28-33-03_84d3000e3f4017145260f7618db1d683

The best option is change your kernel

marchingon12 commented on Mar 19
@benayat yeah, worked for me. i have compiled in my repository (aarch64). I literally spent two days trying to find the cause or solution. found out when @WLFJ said that version 23.0.1 was broken. i had to compile for aarch64 (my architecture) and using of the version docker 'v20.10.23-ce'

For other users out there, thankfully the root-packages repo has downgraded from v23.0 to v20.10.23 64e0fe so you don't need to build it yourself.

martindmtrv commented on Apr 2
When using bridge mode, I am able to access running containers from outside devices but I lose localhost access. (Cannot access the running container from localhost or external ip on the device itself, nor from other containers). Any idea how to resolve it?

When using host mode, I can access the container just fine from the device itself and other containers too. I have run the commands for getting bridge mode to work from the tutorial

Ip route shows this

$ sudo ip route show
default via 192.168.0.1 dev wlan0 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
192.168.0.0/24 dev wlan0 proto kernel scope link src 192.168.0.160
Morakhiyasaiyam commented on Apr 3
@martindmtrv no need to specify host mode or bridge mode if you are connected to wifi then run code I provided earlier in this post or if you cannot find post then go to my repository there you will find

martindmtrv commented on Apr 3
Nice! It worked for me I was missing the last two lines of network script since it isn't mentioned in the main tutorial

These lines are added to my boot script now so loopbacks work and I can reach the containers thanks @Morakhiyasaiyam!

ser646 commented on Apr 4 • 
Can someone help with this error when installing postgres?

PostgreSQL init process complete; ready for start up.
2023-04-03 21:12:15.364 UTC [1] LOG: starting PostgreSQL 15.2 (Debian 15.2-1.pgdg110+1) on aarch64-unknown-linux-gnu, compiled by gcc (Debian 10.2.1-6) 10.2.1 20210110, 64-bit
2023-04-03 21:12:15.365 UTC [1] LOG: could not create IPv6 socket for address "::": Permission denied
2023-04-03 21:12:15.365 UTC [1] LOG: could not create IPv4 socket for address "0.0.0.0": Permission denied
2023-04-03 21:12:15.365 UTC [1] WARNING: could not create listen socket for "*"
2023-04-03 21:12:15.365 UTC [1] FATAL: could not create any TCP/IP sockets
2023-04-03 21:12:15.371 UTC [1] LOG: database system is shut down

I have tried many listen_address configurations, but nothing seems to work. I also have setenforce 0, but still can't get it to work.

rexver213 commented on Apr 4
Can i use x86_64 docker,in an arm64 device?

whyakari commented on Apr 4
Can i use x86_64 docker,in an arm64 device?

there is no way to.

Ilya114 commented on Apr 5 • 
I have error when compiling tini:

Details
SummerOak commented on Apr 11 • 
I have compiled and startup the dockerd, but when I docker run to create a container, an error occur:
POST /v1.41/containers/create returned error: open /data/docker/lib/docker/overlay/8b8c3422356a8893f7a45f4e09c180be1a2d7adfa5b7e63dc30f9279c4e8c9c9-init/merged/etc/hosts: permission denied

The dockerd and dockercli are run as root.
The kernel is 3.18， not support overlay2 , so I set "storage-driver" to "overlay" not "overlay2".
Can someone help with this error, thanks in advance~~

marsonplay commented on Apr 28
When I run sudo docker run hello-world I get the following error, how can I fix it? what am I doing wrong?

INFO[2023-04-27T21:55:28.951207978Z] shim disconnected                             id=8f8518eaaf8552ef3b8ae2084cc3b0ea7ccc617b6eee3507b38501dd42925e13
ERRO[2023-04-27T21:55:28.954178030Z] stream copy error: reading from a closed fifo 
ERRO[2023-04-27T21:55:28.954478238Z] stream copy error: reading from a closed fifo 
ERRO[2023-04-27T21:55:29.123758880Z] 8f8518eaaf8552ef3b8ae2084cc3b0ea7ccc617b6eee3507b38501dd42925e13 cleanup: failed to delete container from containerd: no such container 
ERRO[2023-04-27T21:55:29.124013412Z] Handler for POST /v1.41/containers/8f8518eaaf8552ef3b8ae2084cc3b0ea7ccc617b6eee3507b38501dd42925e13/start returned error: OCI runtime create failed: runc create failed: unable to start container process: error during container init: error jailing process inside rootfs: pivot_root .: invalid argument: unknown 
shaunmulligan commented on May 2
My kernel does not include the CONFIG_AUFS_FS option at all. The kernel stops building after enabling CONFIG_SECURITY_APPARMOR. Unfortunately, I don't have the net/netfilter/xt_qtaguid.c file: Kernel , Sony Xperia XA2 Pioneer $: sudo docker run hello-world # reboots my phone on termux

I found information that from Android 9 and Kernel 4.9 xt_qtaguid is deprecated and replaced with xt_bpf

After compiling the kernel, I have:

# I'm building kernel. On Android via Magisk, I patching boot.img
# $ adb reboot fastboot && fastboot boot ~/kernel/06_magisk_patched-25100_WJlqV.img
# $ adb root; adb shell /data/data/com.termux/files/home/dockerKernel/check-config.sh

Generally Necessary:
- cgroup hierarchy: cgroupv2
  Controllers:
  - cpu: missing
  - cpuset: missing
  - io: missing
  - memory: missing
  - pids: available
  ...
- CONFIG_SECURITY_APPARMOR: missing
...
- Storage Drivers:
  - "aufs":
    - CONFIG_AUFS_FS: missing
...
  - "zfs":
    - /dev/zfs: missing
    - zfs command: missing
    - zpool command: missing
# others options is enabled
On two terminals in Termux via ssh -p 8022 ...:

sudo dockerd                  # or ns && ds aliases run ok in one terminal
sudo docker pull hello-world  # is ok to on the second terminal
sudo docker run hello-world   # this restarts my phone and the logs below
Log docker:

~ $ docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
~ $ docker pull hello-world
Using default tag: latest
latest: Pulling from library/hello-world
Digest: sha256:53f1bbee2f52c39e41682ee1d388285290c5c8a76cc92b42687eecf38e0af3f0
Status: Image is up to date for hello-world:latest
docker.io/library/hello-world:latest
~ $ docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
~ $ docker run hello-world
docker: Error response from daemon: OCI runtime create failed: container_linux.go:370: starting container process caused: process_linux.go:459: container init caused: rootfs_linux.go:59: mounting "mqueue" to rootfs at "/dev/mqueue" caused: device or resource busy: unknown.
ERRO[0000] error waiting for container: context canceled 
Log dockerd:

...
INFO[2022-07-30T19:21:54.727900612Z] Docker daemon                                 commit=aa7e414 graphdriver(s)=overlay2 version=dev
INFO[2022-07-30T19:21:54.731618164Z] Daemon has completed initialization          
INFO[2022-07-30T19:21:54.826863373Z] API listen on /data/docker/run/docker.sock 
INFO[2022-07-30T19:22:27.517670287Z] /etc/resolv.conf does not exist              
INFO[2022-07-30T19:22:27.517791277Z] No non-localhost DNS nameservers are left in resolv.conf. Using default external servers: [nameserver 8.8.8.8 nameserver 8.8.4.4] 
INFO[2022-07-30T19:22:27.517823881Z] IPv6 enabled; Adding default IPv6 external servers: [nameserver 2001:4860:4860::8888 nameserver 2001:4860:4860::8844] 
time="2022-07-30T19:22:27.619591225Z" level=info msg="starting signal loop" namespace=moby path=/data/docker/run/docker/containerd/daemon/io.containerd.runtime.v2.task/moby/4351e7ab82dee1f247e8828c0b389cfef5e98ff8a846915edbd334ed05983d21 pid=14366
Any advice?

Thanks for a good job

@Kris013f did you ever figure this issue out? I am hitting the same thing when trying to run containers on my OnePlus 5T (lineageOS 20.0 with a 4.4 kernel). I have this working perfectly on a Google Pixel 4a (lineageOS 20.0 but with a 4.19) kernel.

shaunmulligan commented on May 3
@camelstrike Yes.docker works well

@R29kTA can you point me to kernel source and toolchain you use? I tried with LineageOS kernel and proton-clang and I can build with all features enabled but CONFIG_MEMCG which errors out with: In file included from ../fs/fs-writeback.c:97: ../include/trace/events/writeback.h:147:7: error: incompatible integer to pointer conversion assigning to 'char *' from 'int' [-Werror,-Wint-conversion] path = cgroup_path(cgrp, buf, kernfs_path_len(cgrp->kn) + 1); ^ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ CC security/keys/request_key_auth.o

You input will be greatly appreciated!

Hey @camelstrike sorry for necro bumping this but was wondering if you ever managed to get around this issue. I am trying to get docker running on a 4.4 kernel and hitting this as well. When CONFIG_MEMCG is left off and i try docker run hello-world it reboots the phone. Did you have that experience too?

shaunmulligan commented on May 7
@camelstrike and @Kris013f how did you manage to run the docker daemon with cgroupv2 ? I have spent a few days trying to get that working, but every build of my 4.4.320 kernel always ends up with v1 enabled for docker 😢

kxxt commented on May 8
I just wrote a blog about how I did it on Lineage OS : https://www.kxxt.dev/blog/self-hosting-services-on-android-phone/

lawal1232 commented on Jun 5
I found it really helpful to learn how to run Docker on Android. However, I recently discovered an even better way to install Docker on my Samsung Galaxy A5. I followed a tutorial on this page https://tutorial-steps.com/install-docker-natively-on-android-phone-and-use-it-as-a-home-server/ , and it provided a more streamlined process with additional features. Nonetheless, I appreciate the effort you put into sharing your knowledge. Keep up the good work!

gas0324 commented on Jun 5 via email 
来信已收到。
lateautumn233 commented 3 days ago
Hello, may I ask if you are interested in Podman?

gas0324 commented 3 days ago via email 
来信已收到。
Add a quote, <Ctrl+Shift+.>
Add code, <Ctrl+e>
Add a link, <Ctrl+k>
Directly mention a user or team
Reference an issue or pull request
Leave a comment
Footer
© 2023 GitHub, Inc.
Footer navigation
Terms
Privacy
Security
Status
Docs
Contact GitHub
Pricing
API
Training
Blog
About
