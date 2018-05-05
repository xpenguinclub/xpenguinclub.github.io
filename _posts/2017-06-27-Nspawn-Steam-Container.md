---
title: "Running Steam in a systemd-nspawn Container"
image: nspawn_featured.jpg
subtitle: "Pack it up, pack it in"
---

Systemd has a cool thing called nspawn which is a mechanism for running things in containers. The [Arch Wiki](https://wiki.archlinux.org/index.php/Systemd-nspawn) puts it thus:

>systemd-nspawn may be used to run a command or OS in a light-weight namespace container. It is more powerful than chroot since it fully virtualizes the file system hierarchy, as well as the process tree, the various IPC subsystems and the host and domain name.
>
>systemd-nspawn limits access to various kernel interfaces in the container to read-only, such as /sys, /proc/sys or /sys/fs/selinux. Network interfaces and the system clock may not be changed from within the container. Device nodes may not be created. The host system cannot be rebooted and kernel modules may not be loaded from within the container.

I will tell the tale of how I created a container in which to run Steam (and other things).

<!--more-->

## Why?

Well, here are *my* reasons:

* It's fun playing with things Linux can do
* To rid my host system of those irritating 32 bit libraries and have a pure 64 bit (host) system
* To isolate proprietary software (Nvidia drivers aside) from the rest of the system
* To prevent Steam and games from spewing clutter into /home

## Caveats

I'm not the most technical person in the world, I may have not done everything in the ideal way. All I can say for sure is: I got it working and here's how. I welcome corrections and improvements (via [pull request](https://github.com/LudicLinux/LudicLinux.github.io), ideally).

This is going to be specific to Arch Linux, pulseaudio and the Nvidia proprietary drivers because that's what I use. It should be pretty easy to adjust it to other set-ups though.

There'll be a few times where I'm going to say I *believe* something to be the case. What this means, exactly, is that what I say is true to the best of my knowledge but I don't know 100% for sure. Stuff where I've read something but not been able to test it, or made a reasonable assumption. Again, I welcome corrections.

This is pretty involved. If you want a *simple* way to sandbox Steam and keep your /home free of its clutter then `firejail` with the `--private=` flag is a *much* easier way of accomplishing that. 

## Introduction

Systemd provides several mechanisms for interacting with containers. The simplest of these is `systemd-nspawn` which can be used to run any valid container from anywhere. If you want the simplest possible solution you could ignore all the stuff where I mess with unit files and just use `systemd-nspawn` to boot your container and then run things manually from that container's command line.

I wanted the most-automated approach so I'm using systemd to start the container as a service and `machinectl` to interact with it.

In order to be able to use `machinectl` to interact with a container by name I believe it must exist under `/var/lib/machines/`. Symlinks *should* work here but there's currently a [bug](https://github.com/systemd/systemd/issues/2001) preventing this. Ideally I'd symlink into a directory in /home but since this doesn't currently work I'm using actual directories in `/var/lib/machines/`.

Nspawn supports various formats for containers, I'm using the simplest approach — actual directories.

When commands start with '#' they should be run as root (or sudo for those who use it), when they begin with '$' they can be run as a normal user.

I'm calling my container 'steamcontainer', if you call yours something else, replace as necessary. My names is Drew and my username is drew. So replace all instances of drew with your username. Unless you are also Drew in which case hello namepal.

## 1. Creating the container

Ok, with all that blabber out of the way, let's get started.

The [Arch Wiki](https://wiki.archlinux.org/index.php/Systemd-nspawn#Create_and_boot_a_minimal_Arch_Linux_distribution_in_a_container) describes this step well, as you'd expect, so I'm really just repeating what it says.

First make the directory where your container will live

    # mkdir /var/lib/machines/steamcontainer

Then install the base of Arch Linux into that container

    # pacstrap -i -c -d /var/lib/machines/steamcontainer base --ignore linux

You can install other distros — the [Arch Wiki](https://wiki.archlinux.org/index.php/Systemd-nspawn#Create_a_Debian_or_Ubuntu_environment) explains how to install Debian and Ubuntu. However, I read in several places that for OpenGL (which, obviously, we need for games) to work, the graphics drivers must match exactly on host and container which I assume precludes running different distros.

We `--ignore linux` because the container doesn't need a kernel, everything runs through the host's kernel. So when the installer asks whether you want to install `linux`, say no. Defaults are fine for everything else.

Once that's done, which should only take a few seconds, we can start the container and install all the other stuff we need.

## 2. Installing all the other stuff we need

Boot the container with

    # systemd-nspawn -b -D /var/lib/machines/steamcontainer

It boots like regular Linux! Pretty cool. When you get over the novelty, log in as root with no password.

I recommend changing root's password now in case you fuck things up and need it at some point, so do that

    # passwd root

Set whatever you like as root's password then proceed.

Edit `/etc/pacman.conf` to enable multilib since we want the 32 bit stuff in the container. Nano will already be installed at this point, use that or install whichever text editor you like and then

    # nano /etc/pacman.conf

And uncomment these lines:

    [multilib]
    Include = /etc/pacman.d/mirrorlist

Refresh the package stuff

    # pacman -Syu

Then let's install all the stuff

    # pacman -S steam steam-native-runtime nvidia-utils lib32-nvidia-utils alsa-utils sudo

Choose whatever fonts you like and select the libgl provider appropriate to your card (the default `libglvnd` is correct for recent Nvidia cards). Proceed with the installation, it should be a coupla hundred megs and shouldn't take long.

`sudo` isn't strictly necessary but it can make things easier later on. Run `visudo` and set it up to your liking, of course. Also install your preferred tools like text editors and so on now.

(I use urxvt as my terminal and had to install `rxvt-unicode-terminfo` to get the shell to behave itself)

Also install a small graphical utility so you can test that X is working correctly. I use `nvidia-settings` since that makes sense to me but you can use any graphical application you like (so long as it doesn't need OpenGL, we're not that far along yet).

## 3. Let's see if X works

Ok now we can test X.

A quick **WARNING** about security. To run graphical applications on the container you have to enable local connections to your X server. This allows anyone on your local network to connect to your X server and potentially do nefarious things. Rather than re-iterating at every point where this is relevant I'll just say: Come up with a solution to this that you're comfortable with. You could manually enable this each time you want to run a game and then disable it once the game is running, you could find a way to automate it or whatever else you like. I'm not going to tell you what to do, it's up to you.

Right, so with that in mind, open a terminal **on the host machine** and open up Xorg to local connections using `xhost` (install `xorg-xhost` if you don't have it)

    $ xhost +local:

While you're there, get the address of your current display

    $ echo $DISPLAY

It'll output something like this

    d@l ~ :) echo $DISPLAY                                      
    :0

Whatever it outputs, use that when you set the `$DISPLAY` environment variable on the container. I'll be using `:0`, replace as necessary.

Then back to the **container** shell and run your test application to see if X stuff is working.

Set that `$DISPLAY` variable we talked about

	# export $DISPLAY=:0

Then run your test application

    # nvidia-settings

(Obviously that can be run as a normal user but, if you've followed this to the letter, we don't have one yet)

Did it open? Good!\|Oh no!

Let's assume it did since if you've done everything correctly so far it really should.

Close your test app and get back to the container's shell. There are a few more things to do before we move on to unit files and all that gubbins.

Steam won't run as root (nor do we want it to) so let's create a normal user for Steam

    # useradd -m -G wheel -s /bin/bash drew

(replace 'drew' with the username you want. Mentioning it here because it's the first time it's come up and you may not have noticed)

That'll add a user with the same primary group as their username and the additional group wheel, which is handy for things like sudo.

    [root@steamcontainer ~]# groups drew
    wheel drew

See?


Set a password for that user

    # passwd drew

Whatever you like.

## 4. The Sound of Sound

Many games hook into ALSA rather than pulseuadio. We need to direct them to pulse so create `/etc/asound.conf`

    # nano /etc/asound.conf

And put this in it

    # Use PulseAudio by default
    pcm.!default {
      type pulse
      fallback "sysdefault"
      hint {
        show on
        description "Default ALSA Output (currently PulseAudio Sound Server)"
      }
    }
    
    ctl.!default {
      type pulse
      fallback "sysdefault"
    }

I don't know what any of that means but it works. If you don't use pulseaudio then you're probably cleverer than me and know what to do here.

While we're here let's do one last thing and set up the environment variables the container instance is going to need to talk to the things it needs to talk to.

Edit `/etc/environment`

    # nano /etc/environment

And add these two lines to the bottom

    DISPLAY=:0
    PULSE_SERVER=unix:/run/user/host/pulse/native

(Adjusting `:0` to the value you got, of course)

Ok, we're done in the container for now so close it down by holding `Ctrl` and hitting `]` 3 times (the documentation says this only works on a US layout but it worked fine for me with my UK layout. If it doesn't work try the key `%` is on instead of `]`).

Let's hop to a shell on the host.

## 5. Units and that

I want to start the container with `systemctl` but we need to override some defaults in order to get OpenGL and sound working. You can either create an override *specific* to the `steamcontainer` container *or* one which applies to all containers started with `systemctl`. Since I'd like to have this ability available to all my (potential future) containers, I'm choosing the latter route. If you'd prefer to do the former then instead of the line below, you could do `# systemctl edit systemd-nspawn@steamcontainer.service`. It's up to you.

Ok, so let's create the override file (on the **host**)

    # mkdir /etc/systemd/system/systemd-nspawn@.service.d/
    # nano /etc/systemd/system/systemd-nspawn@.service.d/override.conf

And put the following in it

    [Service]
    DeviceAllow=/dev/dri rw
    DeviceAllow=/dev/shm rw
    DeviceAllow=/dev/nvidia0 rw
    DeviceAllow=/dev/nvidiactl rw
    DeviceAllow=/dev/nvidia-modeset rw
    DeviceAllow=char-usb_device rwm 
    DeviceAllow=char-input rwm 
    DeviceAllow=char-alsa rwm 
    ExecStart=
    ExecStart=/usr/bin/systemd-nspawn --quiet --keep-unit --boot --link-journal=try-guest --machine=%I

The first five lines after `[Service]` ensure that the container has read/write access to the stuff it needs for accelerated graphics. The next three lines pass input devices (so controllers work) and ALSA devices through (for sound).

The last two lines override the default networking behaviour and just pass the host's network through to the client. The reason I've done this is that when I was using the default configuration, Steam kept saying "There have been too many failed login attempts from this location" (when there had in fact been none). If you'd rather use the default networking behaviour and see if the same happens to you (or fix it!), go for it — omit those last two lines.

Now we're going to create a `.nspawn` file which contains specific runtime settings for particular containers.

You'll notice that some of the `Bind` directives mirror what's in the other file and some don't. This is because normal filesystem binds are, by default, read/write whereas binds to kernel subsystems are, by default, read-only and need to be set to read/write above.

The upshot of that is that you can add as many additional filesystem binds here as you like. I've added my Downloads directory, for example, so that I can access any games downloaded outside of Steam in the container.

Let's make that file

    # mkdir /etc/systemd/nspawn/
	# nano /etc/systemd/nspawn/steamcontainer.nspawn

And put the following in it

    [Exec]
    Boot=1
    
    [Files]
    # nvidia-opengl
    Bind=/tmp/.X11-unix
    Bind=/dev/dri
    Bind=/dev/shm
    Bind=/dev/nvidia0
    Bind=/dev/nvidiactl
    Bind=/dev/nvidia-modeset
    
    # input
    Bind=/dev/input

    # pulseaudio
    Bind=/run/user/1000/pulse:/run/user/host/pulse

    # alsa
    Bind=/dev/snd
    
    # downloads
    Bind=/home/drew/Downloads
    
    # Pacman cache
    Bind=/var/cache/pacman/pkg

That should all be pretty self-explanatory with the comments. It's not necessary to bind Pacman's cache, I've just done that since the host and container may as well share the same cache.

In my container I've got an extra bind which links an external Steam library into the home folder of my user on the container.

## 6. Testing it all out

We can now start the container as a service using `systemctl`. Once it's running we can interact with it using `machinectl` to do things like open a shell or run a command as a particular user. `systemd-run` can also be used to run commands but can't run commands on remote machines (containers count as remote) as normal users, which is why I'm using `machinectl`.

Let's start the container as a service (on the **host**)

    # systemctl start systemd-nspawn@steamcontainer.service

You can obviously put that command in a file that's run at startup to have the container run automatically. I've put it in my `.xinitrc` file. Normally you'd just `enable` the service but for me it always fails because `/dev/nvidia-modeset` seems to be created after any target I've tried, causing the service to fail. Feel free to mess with that and let me know if you find a way to make it work.

You can use

    # systemctl status systemd-nspawn@steamcontainer.service

To check everything's ok if you like.

Ok, so our container is up and running, let's connect to it and see if stuff works. Remember you'll need `host +local:` set on the host machine if you want to run graphical things.

    # machinectl shell drew@steamcontainer

Replace 'drew' with the username of the user you created on the container. You should see a shell for your normal user. Let's run steam

    $ steam

Steam runs! Amazing.

Install some games (or bind an external Steam library in `/etc/systemd/nspawn/steamcontainer.nspawn` and then add it in Steam) and try them out.

You'll need to restart the service between config changes by doing

    # systemctl restart systemd-nspawn@steamcontainer.service

(or `stop` and then `start` instead of `restart` if you prefer)

Once you've got everything to your liking, rather than opening a shell and typing the name of the command you want to run you can use

    # machinectl shell drew@steamcontainer /usr/bin/steam

To start any executable. You can obviously put that command somewhere where it autostarts or add it into a `.desktop` file or a shell script, key combo — whatever method you prefer. You can edit the last bit to run anything you like on the container.

For example, I've moved my Wine prefixes into my container user's home directory (`man machinectl` and see `copy-to` and `copy-from`) and installed Wine on the container so I can completely rid my host machine of 32 bit libraries.

And here it is all working. It looks... like Steam.

{% include img.html image="nspawn_steam.jpg" %}

## Notes

The only thing I've noticed that doesn't work is `steam://` protocol links from web browsers etc.. Presumably because Steam is running on a different 'machine' to the web browser. I've never had any use for those links though so that doesn't bother me.

Thanks to [/u/landen_schmitt](https://www.reddit.com/r/linux_gaming/comments/6jcicl/ive_been_using_systemdnspawn_to_play_games_and_i/) and [/u/lesdoggg](https://www.reddit.com/r/linux_gaming/comments/6jcicl/ive_been_using_systemdnspawn_to_play_games_and_i/djdb84a/) for giving me the idea to do this. 

Image: [Giovanni di Paolo - Saint John the Baptist in Prison Visited by Two Disciples](https://commons.wikimedia.org/wiki/File:Giovanni_di_Paolo_-_Saint_John_the_Baptist_in_Prison_Visited_by_Two_Disciples_-_Google_Art_Project.jpg).
