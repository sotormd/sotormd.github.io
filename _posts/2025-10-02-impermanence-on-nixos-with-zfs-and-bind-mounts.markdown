---
layout: post
title: "Impermanence on NixOS with ZFS and Bind Mounts, without the Impermanence library"
date: 2025-10-02 23:51:11 +0530
categories: nixos zfs impermanence
---

### New Computer Smell

I love it when my boxes are new. The new computer smell is comparable only to
the soothing petrichor after a calming shower.

This fleeting affection lasts for about two days. After that, the stench of
accumulated system state shrouds any remaining freshness.

An ideal system would have no state. It would be exactly the same today as it
was yesterday, and it will be the same tomorrow.

This, however, is impossible. Unless your box is some kind of embedded device.
If your box is your main PC or home server, there are at least five things you'd
like to persist across reboots.

So, if we could somehow keep these "important" things and nuke everything else,
we would be in a constant state of new computer smell.

Remember, if you are not in a constant state of new computer petrichor, you are
in a constant state of old computer stench.

Okay, all this sounds nice on paper. But implementing something like this is
very hard on any traditional Linux distribution. Take Debian, for example.
Nuking the root partition would obviously lead to an unusable system.

So, what are the files we need to persist to have a working system?

- Most of `/bin`
- Most of `/usr/bin`
- Most of `/usr/share`
- Most of `/etc`

I could go on, but you see my point. We are left with very little leeway.

NixOS, on the other hand, requires only `/boot` and `/nix` to boot. Everything
else is symlinked from `/nix` during boot time.

Which is perfect for achieving what we want.

What seemed herculean on Debian is quite trivial here. In fact, there exists a
[nix-community project](https://github.com/nix-community/impermanence) to
address this issue specifically.

However, we can actually implement this ourselves using glorious ZFS and bind
mounts.

### ZFS Datasets

First, we can decide on a basic scheme for our ZFS datasets:

- `rpool/nix` at `/nix`
- `rpool/root` at `/`
- `rpool/home` at `/home`
- `rpool/persist` at `/persist`

…and of course, a FAT32 `/boot` as the ESP.

The plan is simple: every boot, `rpool/root` and `rpool/home` are pulverized.
Everything in `/boot`, `rpool/nix`, and `rpool/persist` will survive the
cataclysm.

This allows us to keep files that require persistence on `rpool/persist` and
bind mount them.

The pulverization will be a simple `zfs rollback` to an empty snapshot.

While creating the datasets, take the empty snapshots so we can roll back to
them:

```console
# zfs snapshot rpool/root@blank
# zfs snapshot rpool/home@blank
```

Now we can proceed with the normal installation process.

Once your box is fully set up, we can decide what we want to persist across
sessions.

### Opt-in Persistence

From the NixOS [manual](https://nixos.org/manual/nixos/stable/#ch-system-state),
these directories are required (apart from `/nix` and `/boot`):

- `/var/lib/nixos`
- `/var/lib/systemd`
- `/etc/zfs/zpool.cache`
- `/var/log`
- `/etc/machine-id`

The `machine-id` can be passed as a kernel parameter, so we can skip that and
set:

```nix
boot.kernelParams = [ "systemd.machine_id=${config.vars.device.machineId}" ];
```

And these are my personal picks:

- `/var/lib/sbctl`, you probably need this if you use Secure Boot.
- `~/Documents`
- `~/Downloads`
- `~/Pictures`
- `~/.ssh`
- `~/.config/BraveSoftware/Brave-Browser`

Now we can write a NixOS module for the entire affair.

First, we create `/persist/root` and copy everything required to this directory.

We can write bind mounts like this:

```nix
fileSystems."/path/to/thing" = {
  device = "/persist/root/path/to/thing";
  options = [ "bind" "x-gvfs-hide" ];
};
```

The `x-gvfs-hide` option ensures that these bind mounts don't appear in your
file manager as separate mounts, which can be annoying.

Instead of repeating several blocks of code, we can create a helper function
that extends `nixpkgs.lib`:

```nix
rec {
  persistMount = source: destination: {
    ${destination} = {
      device = source;
      options = [
        "bind"
        "x-gvfs-hide"
      ];
    };
  };

  persistDirs = root: dirs: {
    fileSystems = builtins.foldl' (acc: dir: acc // persistMount "${root}${dir}" dir) { } dirs;
  };
}
```

And now, we can add add this to our NixOS config:

```nix
{ config, lib, ... }:

lib.mkIf config.vars.device.impermanence.enable (
  lib.persistDirs "/persist/root" [
    # Secure Boot
    "/var/lib/sbctl"

    # Needed by NixOS
    "/var/lib/nixos"

    # Needed by systemd
    "/var/lib/systemd"

    # Needed by ZFS
    "/etc/zfs/zpool.cache"

    # Logs
    "/var/log"

    # Documents, Downloads, Pictures, Projects
    "/home/${user}/Documents"
    "/home/${user}/Downloads"
    "/home/${user}/Pictures"
    "/home/${user}/Projects"

    # OpenSSH keys
    "/home/${user}/.ssh"

    # Brave Browser
    "/home/${user}/.config/BraveSoftware/Brave-Browser"
  ]
)
```

### Rollback to Blank Snapshots

Now we can write systemd services for the rollback.

For `rpool/root`:

```nix
# Rollback /
boot.initrd.systemd.services.rollback-root = {
  description = "Rollback /";
  wantedBy = [ "initrd.target" ];
  after = [ "zfs-import-rpool.service" ];
  before = [ "sysroot.mount" ];
  path = with pkgs; [ zfs ];
  unitConfig.DefaultDependencies = "no";
  serviceConfig.Type = "oneshot";
  script = ''
    zfs rollback -r rpool/root@blank
  '';
};
```

For `rpool/home`:

```nix
  # Rollback /home
  boot.initrd.systemd.services.rollback-home = {
    description = "Rollback /home";
    wantedBy = [ "initrd.target" ];
    after = [ "zfs-import-rpool.service" ];
    before = [ "sysroot.mount" ];
    path = with pkgs; [ zfs ];
    unitConfig.DefaultDependencies = "no";
    serviceConfig.Type = "oneshot";
    script = ''
      zfs rollback -r rpool/home@blank
    '';
  };

  # Setup /home
  systemd.services.setup-home = {
    description = "Setup /home";
    wantedBy = [ "local-fs.target" ];
    after = [ "rollback-home.service" "home.mount" ];
    before = [ "home-manager-myUsername.service" ];
    path = with pkgs; [ coreutils ];
    serviceConfig.Type = "oneshot";
    script = ''
      mkdir -p /home/myUsername/.config
      chown myUsername: -R /home/myUsername
    '';
  };
```

I noticed that Nix doesn't play well when you persist something within `.config`
and the directory doesn't exist. So I added the line to create this directory
after the rollback and set the appropriate permissions.

### Testing

Now we can rebuild our system. Remember that rolling back the datasets is
destructive and will delete everything on `rpool/root` and `rpool/home`, so make
sure that everything important is on `/persist` (or wherever you've mounted
`rpool/persist`).

Do a simple test:

```console
$ touch ~/naughty-file
$ touch /naughty-file
```

And reboot. Boom. The naughty files are gone. New computer smell **forever**.

The argument of "new computer smell" is admittedly superficial, but the idea of
ephemeral systems is useful in many contexts.

How often do you check your Ansible playbooks? Every time you make some minor
imperative ad-hoc change to your server, you are moving away from your
well-defined playbook. Ephemeral systems force you to take note of every change
you make and whether to keep them or not.

In our setup, a `zfs diff` will show all changes made:

```console
# zfs diff rpool/home@blank
# zfs diff rpool/root@blank
```

You can go even further and write your own NixOS options to set up the bind
mounts, so you don't have to rewrite a lot of boilerplate code.

### Further Reading

- [Erase your darlings - Graham C](https://grahamc.com/blog/erase-your-darlings/)
  – This is what initially got me interested in impermanence.
- [NixOS as a server, part 1: Impermanence - Guekka](https://guekka.github.io/nixos-server-1/)
- [Configuring NixOS for Impermanence - Mich Murphy](https://mich-murphy.com/nixos-impermanence/)

### Fin
