# foundercoach-update-test

**Test only. Do not install these packages on a device you use for real practice sessions unless
you are testing the update flow.**

This repository hosts a throwaway release used to test the FounderCoach software-update flow on a
Raspberry Pi 5: the update notice, "Remind Me Later", the verified download and the upgrade with
`apt`. It is not where real FounderCoach releases are published.

The release here is signed with a **test key**. A normal FounderCoach installation does not trust
that key, so it never offers this release unless a tester points the device here on purpose
(steps below).

## What is in release v1.0.1

| File | Purpose |
|---|---|
| `foundercoach_1.0.1_arm64.deb` | A FounderCoach package built from the same code as 1.0.0, with only the version number raised to 1.0.1 |
| `foundercoach-update.json` | The update manifest: version, release-notes link, package URL, size and SHA-256 |
| `foundercoach-update.json.sig` | Ed25519 signature of the manifest, made with the test key |

Devices read `releases/latest/download/foundercoach-update.json`. GitHub's "latest" link skips
pre-releases, so a test release must be published as a normal (latest) release.

## How the update flow works

1. Once a day, when online, the app downloads the manifest and its signature over HTTPS.
2. It checks the signature against the public key installed on the device. A wrong signature,
   invalid content, a pre-release version or a missing arm64 package means nothing is offered.
3. If the version is newer than the installed one, a notice appears in the app with
   **Download Update**, **Remind Me Later** (hides it for 7 days) and **Release Notes**. The
   notice never appears while something is recording, uploading or being analyzed.
4. **Download Update** saves the package to `/var/cache/foundercoach/updates` only if its size and
   SHA-256 match the signed manifest. Nothing is installed automatically.
5. The user installs it with `sudo apt install <path>`. The upgrade is refused while the device
   is busy, and the practice database is backed up first.

## Testing on a Raspberry Pi with FounderCoach 1.0.0 installed

You need the test public key that matches the signature (ask the maintainer; it is not stored
here). The override below leaves the installed settings file untouched and is easy to remove.

```sh
sudo cp test-update-signing.pub /etc/foundercoach/test-update-signing.pub

printf 'FOUNDERCOACH_UPDATE_MANIFEST_URL=https://github.com/lusun1121/foundercoach-update-test/releases/latest/download/foundercoach-update.json\nFOUNDERCOACH_UPDATE_PUBLIC_KEY=/etc/foundercoach/test-update-signing.pub\n' \
  | sudo tee /etc/foundercoach/update-test.env

sudo mkdir -p /etc/systemd/system/foundercoach.service.d
printf '[Service]\nEnvironmentFile=/etc/foundercoach/update-test.env\n' \
  | sudo tee /etc/systemd/system/foundercoach.service.d/update-test.conf

sudo rm -f /var/lib/foundercoach/update-state.json
sudo systemctl daemon-reload && sudo systemctl restart foundercoach
```

After a few seconds:

```sh
curl -s http://127.0.0.1:8000/api/app/update | python3 -m json.tool | head -12
```

`"status": "update_available"` with version 1.0.1 means the device found and verified the
release. Open the FounderCoach app to see the notice.

### Checklist

| Check | How | Expected |
|---|---|---|
| Notice | Open the app | "FounderCoach 1.0.1 is available" at the top of the page |
| Release Notes | Click it | Opens the v1.0.1 release page here |
| Remind Me Later | Click it, then reload or switch pages | Notice stays hidden for 7 days |
| Show it again | `sudo rm /var/lib/foundercoach/update-state.json && sudo systemctl restart foundercoach` | Notice is back |
| Busy | Start a recording and wait about a minute | Notice disappears and returns after the recording |
| Download | Click **Download Update** | "Downloaded and verified" and the install command |
| Not auto-installed | `dpkg -l foundercoach` | Still 1.0.0 |
| Upgrade | `cd ~ && sudo apt install /var/cache/foundercoach/updates/foundercoach_1.0.1_arm64.deb` | Version 1.0.1, a new backup in `/var/lib/foundercoach/backups`, history intact |
| Rollback | `cd ~ && sudo apt install ./foundercoach_1.0.0_arm64.deb` and confirm the downgrade | Version 1.0.0, history intact |
| Offline | Disconnect the network, clear the state file, restart | `"status": "offline"`, no notice, the app keeps working |

### Clean up afterwards

```sh
sudo rm -f /etc/systemd/system/foundercoach.service.d/update-test.conf \
  /etc/foundercoach/update-test.env /etc/foundercoach/test-update-signing.pub \
  /var/lib/foundercoach/update-state.json
sudo systemctl daemon-reload && sudo systemctl restart foundercoach
```

If the device ended on 1.0.1, reinstall the real 1.0.0 package: the 1.0.1 package here carries
the test key.

## Status

Tested on a Raspberry Pi 5 (Raspberry Pi OS 64-bit, Debian 13) on 2026-11-16: the release assets
verified, and the update notice, reminder and update checks passed.

The FounderCoach source and full documentation (install, update, rollback and the release
process) are in the private `altitut/alti-on-device-ai` repository, branch
`feat/pi-local-app-updates`: `README.md` and `docs/pi_desktop_app.md`.
