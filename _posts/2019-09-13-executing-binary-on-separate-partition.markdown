---
layout: post
title: Executing Binary File on Separate Partition
date: '2019-09-13 07:04:00'
image: assets/images/green_binary_linux_tux_wallpaper_by_m0gria_d4zibzq-fullview.jpg
categories: [linux]
---
So today i tried to install [gotour](https://tour.golang.org) offline, because sometimes i just don't get internet connection, and it's convinient to run it offline. How to get it ? just run simple 

~~~.language-bash
go get golang.org/x/tour
~~~

It will install gotour in your ```$GOPATH``` directory. Then go to ```$GOPATH/bin``` then run your binary

~~~.language-bash
./tour
~~~

It should run [tour.golang.org](tour.golang.org) on your local machine, but in my case, 

~~~.language-bash
./tour
zsh: permission denied
~~~

What was that ?

After some browsing on internet, it turns out that executing binary from separate partition isn't allowed by linux. Because my GOPATH was on separate partition, then i can't do it. This is actually some kind of security mechanism to preventing people from executing random stuff on your machine, perhaps.

So, how to find it out ? run below command

~~~.language-bash
▶ mount | grep noexec
~~~

The output would be something  like this

~~~.language-bash
...
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
~~~

This is list of partition that has ```noexec``` properties when they mounted. If you see your partition there, then you can't execute stuff from there.

Temporary fix would be

~~~.language-bash
▶ sudo mount -o remount,exec /your/mount/point
~~~

This would remount your partition with exec properties, but this configuration would reset when rebooted. *Note that somehow this method doesn't work for me, dunno why

A permanent fix would be editing the ```/etc/fstab``` file, and add exec properties at the end of line

~~~.language-bash
UUID=69696xxx /media/username/ ntfs errors=remount-ro,auto,rw,user,exec
~~~

Note that the order of ```exec``` properties is also important, put it at the end of the line to be safe, or make sure it is placed after ```user``` properties, because it is by default applying ```noexec``` property.

We're done. See you on next post

Special thanks for the folks at askubuntu [here](https://askubuntu.com/questions/678857/fstab-doesnt-mount-with-exec)







