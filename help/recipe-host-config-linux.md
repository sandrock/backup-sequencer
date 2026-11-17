host-config-linux recipe
================================

Backup of a Linux host's own configuration: what you need to rebuild the machine
itself, not the services it runs (each service is its own sequencer entry).
Bundled at `recipes/host-config-linux.ps1`. Nothing is stopped: no downtime.


What it does
----------------------------------------------------------------

One step, `dump`, which produces:

- `<service>_files.tar` — the standard configuration locations, stored relative to
  `/` (`etc/fstab`, `root/.bashrc`, …):
  - `/etc`
  - `/var/spool/cron/crontabs` (user crontabs, root's included)
  - `/usr/local/etc`
  - `/root` (minus `.cache`)
  - plus any dir listed in `HCL_EXTRA_DIRS`

  Default locations missing on a host are skipped.
- `facts/` — one text file per command describing the machine. These are not
  restored; they are the checklist for rebuilding by hand:

  | File | Content |
  |---|---|
  | `uname`, `hostnamectl` | kernel, OS, hostname |
  | `dpkg-selections`, `apt-manual`, `apt-policy` | installed packages, manually installed ones, apt sources |
  | `lsblk`, `blkid`, `fdisk`, `df`, `findmnt` | disks, partitions, filesystems, mounts |
  | `mdadm`, `zpool-status`, `zfs-list` | software RAID and ZFS |
  | `ip-addr`, `ip-route`, `iptables`, `nft` | network and firewall rules |
  | `systemd-enabled`, `systemd-timers` | enabled units and timers |
  | `docker-version`, `docker-images`, `docker-networks`, `docker-compose-ls` | docker |
  | `usr-local` | every file in `/usr/local/{bin,sbin}` with size, date and sha256 |
  | `_skipped.txt` | commands not installed on this host |

`/usr/local/bin` and `/usr/local/sbin` are listed, not archived: they mostly hold
downloaded tools (restic, compose, …) that you reinstall on a rebuild. Keep your own
scripts in a dir you list in `HCL_EXTRA_DIRS`.

A command that is not installed is skipped. One that fails or runs over 60 s keeps
whatever it printed, plus its exit code and error output, in its file; the run shows
one warning and the backup still succeeds. tar's "file changed as we read it" is a
warning too (files under `/etc` can change during a live copy).


Secrets
----------------------------------------------------------------

Secrets are always left out — a rebuilt server gets new ones. There is no switch to
include them. Reinstalling the same machine therefore gives it new ssh host keys
(clients will warn once).

Left out by path:

- passwords: `/etc/shadow*`, `/etc/gshadow*`, `/etc/security/opasswd`
- host and TLS keys: `/etc/ssh/ssh_host_*_key` (the `.pub` files are kept),
  `/etc/ssl/private`, `/etc/pki/tls/private`, `/etc/letsencrypt`
- VPN and network: `/etc/wireguard`, `/etc/openvpn`, `/etc/ipsec.secrets`,
  `/etc/swanctl/private`, `/etc/NetworkManager/system-connections`, `/etc/wpa_supplicant`
- directory and auth: `/etc/krb5.keytab`, `/etc/ldap.secret`, `/etc/samba/smbpasswd`,
  `/etc/sssd`
- service credentials: `/etc/postfix/sasl_passwd*`, `/etc/msmtprc`, `/etc/ssmtp`,
  `/etc/ddclient.conf`, `/etc/mysql/debian.cnf`, `/etc/davfs2/secrets`,
  `/etc/apt/auth.conf*`, `/etc/ceph/*.keyring`, `/etc/credstore*`
- the sequencer's own: `/etc/backup-sequencer/secrets`, and the file named by
  `RESTIC_PASSWORD_FILE`

Left out by name, anywhere in the archived dirs:

- key files: `*.key`, `*.p12`, `*.pfx`, `*.keytab`, `*.keyring`, ssh private keys
  (`id_rsa`, `id_ecdsa`, `id_ed25519`, …)
- credential files: `.netrc`, `.pgpass`, `.my.cnf`, `.git-credentials`,
  `.smbcredentials`, anything named `*credentials*`
- tool logins: `.docker/config.json`, `.config/rclone`, `.aws`, `.azure`, `.kube`,
  `.gnupg`, `.local/share/keyrings`
- shell histories (`.*_history`, `.lesshst`): passwords typed on a command line
  end up there

Kept: public certificates (`/etc/ssl/certs`, `*.pem` files), `authorized_keys`,
`known_hosts`.

**Not caught: a secret written inside an ordinary config file** — an `fstab` mount
with `user:password`, a password on a cron line, a Wi-Fi key in a netplan file, a
token in `/etc/default/*`. The recipe cannot remove one line from a file. These end
up in the archive (encrypted when the archiver is restic). Either exclude the file
with `HCL_EXCLUDES`, or move the secret to a separate credentials file (for
`fstab`: `credentials=/root/.smbcredentials`, which is then left out).


Configuration
----------------------------------------------------------------

```json
"host-config": {
  "enabled": true,
  "script": "recipes/host-config-linux.ps1",
  "env": {
    "HCL_EXTRA_DIRS": "/opt/admin-scripts"
  },
  "publishToReady": false,
  "syncArchive": { "script": "recipes/archive-restic.ps1" },
  "steps": [ { "name": "dump", "resources": [] } ]
}
```

| Variable | Meaning | Default |
|---|---|---|
| `HCL_EXTRA_DIRS` | more absolute dirs to archive, `;`-separated | none |
| `HCL_EXCLUDES` | more to leave out, `;`-separated. Starting with `/`: that exact path. Otherwise: a name pattern matched anywhere (`*.bak`, `.cache`) | none |
| `HCL_FILES_ARCHIVE_NAME` | name of the files tar | `<service>_files.tar` |
| `HCL_ROOT` | read the files under another root (a mounted system image) | `/` |

`validate` checks that the host is Linux, `tar` is installed, and every
`HCL_EXTRA_DIRS` entry exists. The run needs root (as every `bseq run`): most of
`/etc` and `/root` are not readable otherwise.

The whole thing is small (a few MB) and changes little, so it suits a nightly run;
restic stores only what changed.


Restoring
----------------------------------------------------------------

Do **not** extract the files tar over `/` on a new machine: its `/etc` belongs to
the new install (other disks, UUIDs, package versions). Extract it aside and take
what you need:

```sh
mkdir /tmp/old-host && tar -xf host_files.tar -C /tmp/old-host
diff -ru /tmp/old-host/etc/ssh/sshd_config /etc/ssh/sshd_config
```

Typical order on a new machine: read `facts/` (packages, disk layout, network,
firewall), install the packages from `apt-manual`, reinstall the tools from
`usr-local`, then bring over config files one by one, crontabs last.
