# autonfsutil

`autonfsutil` configures macOS NFS automounts from a JSON file. Shares are
mounted on first access and unmounted by `automountd` when idle.

It uses the Python and system tools included with macOS; no third-party Python
packages are required. `setup` uses `sudo` to update the system automount
configuration.

## Use

Copy the example configuration and edit the server names, export paths, and
mount options:

```sh
mkdir -p ~/.config/autonfsutil
cp mounts.json.example ~/.config/autonfsutil/mounts.json
$EDITOR ~/.config/autonfsutil/mounts.json
```

Then install or refresh the automount configuration and check the shares:

```sh
./autonfsutil setup
./autonfsutil status
```

The default config path is `~/.config/autonfsutil/mounts.json`; override it
with `--config PATH`. `setup` creates the configured base path using a
synthetic symlink, writes `/etc/auto_autonfsutil`, adds its direct-map entry
to `/etc/auto_master`, and installs a LaunchDaemon that restores that entry
after macOS upgrades.

The default mount base is `/nfs`, so a share appears at
`/nfs/<host>/<share>`. See [`mounts.json.example`](mounts.json.example) for
the config structure: `base` is the top-level mount directory, and each entry
under `hosts` names a server, its `mount_nfs` option string, and the shares
to mount as `<share>: <remote export path>`.

## Mount options

The `options` string is passed to `mount_nfs` as-is; see `man mount_nfs` for
the full list. The pitfalls below are quirks of NFS and the macOS NFS
client, not of this tool — they apply to any NFS mount on macOS:

- Servers reachable only via link-local IPv6 (e.g. a VM on a host-only
  interface) need `proto=tcp6`. Without it, automountd probes the server
  over IPv4 only and gives up without sending a single packet.
- Whenever the server cannot reach the client's callback address (NAT, a
  VPN, or link-local IPv6), granted NFSv4 delegations make operations
  hang — on macOS 27.0.1, v4.1 writes over link-local IPv6 wedge in
  uninterruptible sleep on the first write. Use NFSv4.1 plus `nocallback`,
  or NFSv3, which has no callbacks.
- Large `rsize`/`wsize` buffers combined with `rdirplus` suit high-latency
  links, but on older macOS versions (before NFSv4.1 support) this
  combination was seen panicking the NFS client at local-link speeds (not
  reproduced on macOS 27.0.1). NFSv3 with 128K buffers is a safe choice
  for nearby servers.
