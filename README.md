# Connect chroot to a Mac over SSH

This guide walks through the workflow that connects
[chroot](https://github.com/fitzyracing1/termux-app), Joshua Almeida's
Termux fork for Android, to a Mac over SSH. The same steps worked on
26 November 2026: turn on Remote Login on the Mac, install OpenSSH in
chroot, make an Ed25519 key, copy it to the Mac once, and then `ssh`
in.

This repository starts with a short README on `main`. The complete
walkthrough lives on the `docs/workflow` branch until the pull request
is merged.

## Contents

- [Install chroot](#install-chroot)
- [Prepare the Mac](#prepare-the-mac)
- [Make a key on the phone](#make-a-key-on-the-phone)
- [Connect](#connect)
- [Away from home](#away-from-home)
- [Optional macserve](#optional-macserve)
- [Troubleshooting](#troubleshooting)
- [License](#license)

## Install chroot

chroot is a fork of Termux at
<https://github.com/fitzyracing1/termux-app>. The launcher label is
**chroot**. The Android package id stays `com.termux`, so Termux
plugins and the shared user still work. Because the package id is
unchanged, this is not a second app alongside a store-installed
Termux; installing the fork uses the same package.

The `feat/grok-command` branch, in fork-only
[PR #1](https://github.com/fitzyracing1/termux-app/pull/1), added a
built-in `grok` command. That command runs xAI's official Grok Build
`linux-aarch64` binary under termux-chroot. Nothing was opened against
`termux/termux-app`. PR #1 is now merged into the fork's `master`.

A debug preview APK named `chroot-arm64-v8a.apk` was prepared under
the tag `v0.118.0-chroot-preview` on the fork. Checked with the GitHub
API on 26 November 2026, that release is still a draft, so it does not
appear on the public
[Releases](https://github.com/fitzyracing1/termux-app/releases) page.
Once it is published, download `chroot-arm64-v8a.apk` from that page,
open the file on the phone, and allow installing from the browser or
Files app. Until then, build from the fork's `master` (which already
includes the chroot label and the built-in `grok` command), or use a
copy of the preview APK you already have.

## Prepare the Mac

On the Mac, open **System Settings > General > Sharing** and turn
**Remote Login** on.

In Terminal on the Mac, print your account name and the Wi-Fi LAN
address:

```bash
whoami
ipconfig getifaddr en0
```

`en0` is usually Wi-Fi. If that command prints nothing, try `en1`, or
run `ifconfig` and use the `inet` address on the Wi-Fi interface.

Treat the account name as `<user>` and the address as a LAN IP such as
`192.168.x.x`. You will use those values from the phone. This guide
never uses a real username, hostname, or IP address.

The phone and the Mac must be on the same Wi-Fi for the steps below.
To reach the Mac from another network, see [Away from
home](#away-from-home).

## Make a key on the phone

Open chroot on the phone. Install OpenSSH if it is not already
present, then create an Ed25519 key with no passphrase and save it in
the default path so later SSH commands can find it:

```bash
pkg install openssh
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
```

The `-f` and `-N ""` flags matter. They skip the file-name and
passphrase prompts. If you leave those flags off and paste the next
command at a prompt, the key is saved under the wrong name. See
[Troubleshooting](#troubleshooting) if that already happened.

## Connect

Still in chroot, copy the new public key to the Mac. Replace the
placeholders with the username and LAN IP from the Mac:

```bash
ssh-copy-id <user>@192.168.x.x
```

If SSH asks whether you want to continue connecting, type `yes` and
press Enter. Then enter the Mac account password once. After the key
is installed, later logins from this phone should not ask for that
password.

Connect:

```bash
ssh <user>@192.168.x.x
```

A successful login drops you into the Mac's shell (zsh by default).
You can now run commands on the Mac from the phone.

## Away from home

SSH to a LAN address such as `192.168.x.x` only works while the phone
and the Mac are on the same local network. To reach the Mac from
somewhere else, install [Tailscale for
macOS](https://tailscale.com/download/mac) on the Mac and the
[Tailscale Android app](https://tailscale.com/download/android) on the
phone, then sign both devices into the same tailnet.

After both are online, SSH to the Mac's Tailscale IP instead of the
LAN IP:

```bash
ssh <user>@<tailscale-ip>
```

You can read the Tailscale address from the Tailscale app on either
device, or from `./macserve status` if you use the tool in the next
section.

## Optional macserve

[macserve](https://github.com/fitzyracing1/macserve) is a small tool
that sets a Mac up as an SSH server. It can turn on Remote Login, lock
SSH down to key-only authentication, keep the Mac awake on power, add
a phone's public key, report status (including LAN and Tailscale
addresses and low-disk warnings), and undo every change it recorded.

The tool landed through
[PR #1](https://github.com/fitzyracing1/macserve/pull/1) and is now
merged on `main`. After you clone the repository on the Mac:

```bash
git clone https://github.com/fitzyracing1/macserve.git
cd macserve
./macserve --dry-run setup
```

`--dry-run` prints what would happen and does not change anything.
When you are ready to apply the setup:

```bash
./macserve setup
```

Other commands:

```bash
./macserve add-key ~/.ssh/id_ed25519.pub
./macserve status
./macserve undo
```

`setup` and `undo` of system settings need an administrator account
on the Mac. The tool asks before each change that needs `sudo`, unless
you pass `--yes`.

## Troubleshooting

### The key was saved under a strange name

`ssh-keygen` asked for a file name, and the next command was pasted
into that prompt. The key then lives under that odd name (for example,
whatever command you pasted) in the current directory, plus a matching
`.pub` file. `ssh` will not find it in `~/.ssh/id_ed25519`.

Press Ctrl+C to cancel if `ssh-keygen` is still running. In chroot,
that is the CTRL key on the extra keys row, then C.

Delete the stray files and remake the key with explicit flags so the
prompts never appear:

```bash
rm -f "./the-odd-name" "./the-odd-name.pub"
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
```

Replace `the-odd-name` with the file name `ssh-keygen` actually
printed, such as a command you pasted by mistake.

### Passphrases do not match

If you run `ssh-keygen` without `-N` and the two passphrase prompts
do not match, it asks again. Do not paste other commands into those
prompts. Either type the same passphrase twice, or cancel with Ctrl+C
and remake the key with `-N ""` for no passphrase, as in the command
above.

### zsh: locking failed ... no space left on device

A message such as `zsh: locking failed for .../.zsh_history: no space
left on device: reading anyway` means the Mac's disk is full, or close
enough that zsh cannot update its history file. Free some space on the
Mac before you rely on it as a server. App caches and old developer
build leftovers are often safe to clear; leave Downloads and personal
files alone unless you have chosen what to delete.

## License

MIT. Copyright (c) 2026 Joshua Almeida.
