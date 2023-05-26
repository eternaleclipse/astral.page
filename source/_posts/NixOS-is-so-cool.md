---
title: NixOS is so cool!
tags:
- tech
- linux
---

Last night I played with NixOS (huge thanks to my friend [Tzlil](https://tzlil.net) for the recommendation!) and I had so much fun! It's a Linux distro, somewhat similar to Arch Linux in terms of minimalist philosophy but with emphasis on reproducibility.

Keep in mind I'm still a newbie so there is a lot I might be missing in this post! Be sure to check out the (unofficial, community-maintained) [NixOS Wiki](https://nixos.wiki/).

### Reproducibility
This basically means that anything you install is automatically added to your nix file(s). You can easily control your entire system configuration - bootloader, partition layout, wpa_supplicant, packages, user profiles, services, environment, software-specific configuration, **Everything** can be declared in the Nix config files. They freakin' support stuff like chromium / firefox profiles and [extensions](https://search.nixos.org/options?channel=unstable&show=programs.chromium.extensions&from=0&size=50&sort=relevance&type=packages&query=programs.chromium), oh-my-zsh theme, `.vimrc` options, you name it.

These config files can be backed up in a git repository that you can easily deploy whenever you need to restore, clone or modify your system.

### Installation
Installation is super straightforward for what you'd usually expect from this type of minimalistic OS. There is a friendly GUI that lets you pick which type of Desktop Environment you want (if any) and the usual stuff (region, user / pass).

![](/images/nixos-installer.png)

I chose GNOME as my Desktop Environment. After the setup finished, I was greeted by this beautiful screen:

![](/images/nixos-installed.png)

The disk usage for the default GNOME installation is just shy of 9 GB.
```
[user@nixos:~]$ df -h | grep sda
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        196M     0  196M   0% /dev
tmpfs           2.0G  8.0K  2.0G   1% /dev/shm
tmpfs           976M   11M  965M   2% /run
tmpfs           2.0G  456K  2.0G   1% /run/wrappers
/dev/sda1        20G  8.7G  9.9G  47% /
tmpfs           391M  116K  390M   1% /run/user/1000
/dev/sr0        2.3G  2.3G     0 100% /run/media/user/nixos-gnome-22.11-x86_64
```

### Enabling experimental Nix features
**Nix** is NixOS's package manager, it uses it's own expression-based pure-functional language, "Nix", to define the logic that decides which packages are installed. The first thing we wanna do is go ahead and enable "experimental" Nix features. This gives us access to stuff like [Flakes](https://nixos.wiki/wiki/Flakes) which are the new way of creating and managing packages.

### Trying out packages temporarily

If we just wanna play with a package, we can have it downloaded and run temporarily.
When the command completes, the package will disappear from our system!

```
[user@nixos:~]$ echo 123 | nix run nixpkgs#ripgrep 123
123

[user@nixos:~]$ rg
rg: command not found
```

### nix-shell

Using the [nix-shell](https://nixos.org/manual/nix/stable/command-ref/nix-shell.html) command, we can create a temporary, isolated shell environment where we can temporarily play with a package.

```
[user@nixos:~]$ nix-shell -p wget
this path will be fetched (0.62 MiB download, 3.25 MiB unpacked):
  /nix/store/biaz74lcslins8c0kl5pv3yzxp1b02qj-wget-1.21.3
copying path '/nix/store/biaz74lcslins8c0kl5pv3yzxp1b02qj-wget-1.21.3' from 'https://cache.nixos.org'...

[nix-shell:~]$ wget
wget: missing URL
Usage: wget [OPTION]... [URL]...

Try `wget --help' for more options.

[nix-shell:~]$ exit
exit

[user@nixos:~]$ wget
wget: command not found
```

You can also use `nix-shell` with some Nix scripting to make [your own isolated development environment](https://nixos.wiki/wiki/Development_environment_with_nix-shell). How cool is that!

### Installing packages persistently
I'm feeling like typing stuff, so let's install Vim!
To add a package persistently, we can edit our `/etc/nixos/configuration.nix` and add it here:
```
environment.systemPackages = with pkgs; [
  # Existing packages...
  vim
];
```

Then, we need to run `sudo nixos-rebuild switch` and it applies our new configuration, installing the package.

Alternatively, we can simply run `sudo nix-env -iA nixos.vim`.
It actually does the same thing behind the scenes - it changes the `configuration.nix` file.

### Why this excites me
I have a problem with system bloat.
It's not so much that I mind some of my disk being consumed by gigabytes of pointless cyber-garbage.

It's more about the fact that you're constantly afraid to change things, because systems become slow and break over time. Sure, there are backups, but is everything really guaranteed to be there and work correctly when something breaks? This has a terrible consequence - You don't experiment on your host environment, and you don't improve. You don't even know what your system is running under the hood because you're too afraid to look down there! So, how does it do you any good to have your OS entirely open-source if you never change anything because you're afraid to break stuff? 

Having the entire setup configuration declared in a bunch of files in a git repo just frees me to experiment and try crazy stuff, experiment! Try a new WM, mess with the configs, change the kernel! This is the true spirit of open source. If anything ever breaks, slows down or even gets compromised, I just copy over a bunch of files and do a `sudo nixos-rebuild switch`, and if I really mess up, I could always reinstall NixOS and apply the configuration, or even [create an installation ISO based on my config files](https://nixos.wiki/wiki/Creating_a_NixOS_live_CD) to get everything back and working.

If you like a certain setup, you could easily clone it to another PC, or even share it with friends. You could have an entire organizational network definition, including desktops, servers, routers all declared in a central repository, and [deploy](https://nixops.readthedocs.io/en/latest/overview.html) them with a single command, or even have them [PXE boot from network](https://nixos.wiki/wiki/Netboot) and install a fresh config. This level of control and transparency makes maintenance, upgrades, security and life in general so much simpler and more fun.

### References
- https://nixos.org/
- https://nixos.wiki/
- https://fasterthanli.me/series/building-a-rust-service-with-nix/part-9
