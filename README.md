# Running Docker container w/o Docker

# Intro

The goal for this experiment was to take a Docker container and run it in your
favourite linux distro. Experimenting was worth it and for the first time in a
long time I even wrote notes on the way. So, here we go!


# Copy-paste the target container
```
docker run -it --rm --name ubuntu_cont --net=host -v $PWD/workspace:/workspace
ubuntu /bin/bash
```

```
tar -cvf ubuntu_root.tar bin boot dev etc home lib lib64 media mnt opt root run
sbin srv usr var
```
As you can see I chose to copy most of the container and excluded only tmp,
/sys and /proc directories. Both of the directories are mostly mounted proc and
cgroup items so I figured to try the experiment without them. If you are
interested on the mounts and different types of items mounted you can inspect
mount-command and get a deeper look.


Next you should move the file to your target host and unpack the contents to
"/ubunturoot" directory.

My host OS is Centos7 running inside VirtualBox. Transferring the file is easy
with scp and operating inside the host OS is done with ssh.
```
mkdir -v /ubunturoot
cp ubuntu_root.tar ubunturoot/
cd ubunturoot
tar -xvf ubuntu_root.tar

```
# Enter back in
Now we have the containers insides copied in "/ubunturoot" and we can enter our
partially unshared namespaces in a new root directory. After the next command
you will find yourself inside a chroot environment where PID and network
namespaces are isolated from the host OS.

```
unshare --fork --pid --mount-proc chroot /ubunturoot /bin/bash
```

The first thing you will notice is too limited path variable. Update the path
by exporting.
```
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Now you can start exploring around the environment. Try looking at what
processes are running inside your jail with "ps".
```
ps aux
```
Ok, ps does not work since we have not copied the /proc directory and mounted
our OS proc on it. To get a view on the processes you should create the
directory and mount accordingly.

```
mkdir -v /proc
mount -t proc proc /proc
```
ps command just became a few notches more interesting.

We also create a tmp directory with right privileges so that processes using
temporary files can work correctly.

```
mkdir -v /tmp
chmod 777 /tmp
chmod +t /tmp
```

# Add basic networking
To have more fun with your container you might want to have a connection to
outside world. If you try to "apt-get update" you will see that updating is not
possible at the moment. The reason is an outdated "/etc/resolv.conf" file. We
copied this file with our grand copy-paste, but it reflects a wrong
environment.

Update the file to include _only_ the following line.
```
nameserver <host_os_resolv_ip>
```
Find your host OS resolving address and replace it with <host_os_resolv_ip>. As
the environment is pretty bare editing a file can be a bit troublesome. Try out
editing the file like this.
```
echo "nameserver 192.168.43.1" > /etc/resolv.conf
```
Updating apt-get just became possible. Be amazed with apt-get updating.

```
apt-get update
```
As apt-get updated itself in a way you just became self-sufficient in your
little jail. You can for example continue by installing iproute2 package to
inspect your network view of the world and Vim to have more interactive editing
experience.
```
apt-get install -y iproute2 vim
```

# Advanced networking
Having a separete network for you container is important if you want to offer
services from inside your container. To enable networking you should start the
isolated environment with a separate network namespace. To do that we start out
a bit differently.
```
unshare --net --fork --pid --mount-proc chroot /ubunturoot /bin/bash
```
Now we have network isolation added. To connect our container to outside world
we will use virtual Ethernet (vEth) pairs. In this post we will look how we can
insert another of the vEth pairs inside the container, but a more detailed
routing and IP management is left out for another post. Nonetheless, if you
know how to add an interface inside a container you should be able to figure
out the rest from generig vEth tutorials.

Fist we create a vEth pair. On the host type:
```
ip link add vethCont type veth peer name vethHost
```
Set both interfaces UP from DOWN.
```
ip link set vethCont up
ip link set vethHost up
```
To attach the interface inside the container we need PID for the bash command
we are running in our environment. To figure the PID out I'd recommend using ps
in tree mode on the host.
```
ps axjf
```
Find our unshare command and use the child bash process PID. In my case it was
24915.
```
ip link set vethCont netns 24915
```

Look what the network looks like inside the container.
```
ip a
```
You should see an interface by the name "vethCont". As you noticed that the
iproute2 package persisted to our networking part of the experiment since we
did not update our "/ubunturoot" between the sets.

Figured that it should be possible, but not sure how it worked. At this point I
came across a blog about similar setup with the missing command:

Figuring out that you can access namespaces with PID took quite a while to
realize. At first I tried to attach the unshare command to an existing
namespace, but it proved to be more complicated than the PID alternative.

# Conclusion

The experiment started with an interest on namespaces and Linux in general. As
any good experiment this one left many more questions than answers behind.
Especially interesting is the way how you can start separating the OS into
layers and then start manipulating different aspects of the environment.

Next I would like to take a look into SELinux and cgroup aspects of
virtualization in Linux. Another interesting thingy would be to start sketching
and experimenting how Linux can be used to offer a "Function As A Service"
-styled platform.

