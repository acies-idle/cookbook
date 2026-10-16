# OCI runtime error Podman 6 on Windows: crun: controller `pids` is not available

## Configure the machine from the host

## Problem this solves

`astro dev start` (and even `podman run hello-world`) failed with:

```
crun: controller `pids` is not available under /sys/fs/cgroup/non-systemd/...
OCI runtime error
```

Cause: the Podman machine used the `systemd` cgroup manager, which needs a systemd
user session that WSL does not provide. The working setup (Podman 5.x) used `cgroupfs`.

## What changed in Podman 6

Since Podman 6, the machine mounts the host user's containers config folder into the VM:

| Side          | Path                                 |
|---------------|--------------------------------------|
| Windows host  | `C:\Users\<user>\.config\containers` |
| Inside the VM | `/etc/containers`                    |

The release notes give `~/.config/containers` on Linux as the example.

- Podman 5 wrote its own `/etc/containers/containers.conf` inside the VM (with `cgroupfs`).
- Podman 6 mounts the host folder over `/etc/containers`, so that file is no longer there
  unless you create it on the host. Without it, the default `systemd` manager applies.

You edit config files on Windows and the VM sees them.

## How to apply

### 1. Create the config file on Windows (PowerShell)

```powershell
    New-Item -ItemType Directory -Force "$HOME\.config\containers" | Out-Null
    @'
    [engine]
    cgroup_manager = "cgroupfs"
    '@ | Set-Content -Encoding ascii "$HOME\.config\containers\containers.conf"
```

If the file already exists, open it and add the lines instead of overwriting it:

```powershell
notepad "$HOME\.config\containers\containers.conf"
```

### 2. Restart the machine

```powershell
podman machine stop astro-machine
podman machine start astro-machine
```

### 3. Verify

```powershell
podman info | findstr cgroupManager
podman run hello-world
```

`cgroupManager` should show `cgroupfs`.

### OPTIONAL CHECK - Check that the mount reached the VM

```powershell
podman machine ssh astro-machine "ls -la /etc/containers"
```

Your `containers.conf` should be listed. If it is not, the mounted Windows folder is a
different path than `$HOME\.config\containers`.

## Notes

- The file lives on the host, so it survives `podman machine rm` and recreation.

- Do not write to `/etc/containers/containers.conf` from inside the VM on Podman 6. That folder is now the host mount.

- After reinstalling Podman, the VM image also ships
  `/usr/share/containers/containers.conf.d/999-podman-machine-wsl-cgroupfs.conf` (`cgroup_manager="cgroupfs"`). With
  that in place the host file is redundant but harmless.

- Files in the host folder take priority over the defaults in `/usr/share/containers`.

- Other files can be managed the same way, for example `registries.conf` for a corporate registry.
- Add `[network]` `firewall_driver = "iptables"` only if you get netavark or firewall errors. Older Podman 5 machines
  set it as an nftables-on-WSL workaround.

## Comparison that led to this

|                                         | Failing machine | Working machine         |
|-----------------------------------------|-----------------|-------------------------|
| Podman                                  | 6.0.2           | 5.8.x                   |
| cgroupManager                           | systemd         | cgroupfs                |
| `/etc/containers/containers.conf` in VM | missing         | `cgroupfs` + `iptables` |
| WSL / kernel                            | identical       | identical               |
| Rootful                                 | false           | false                   |

