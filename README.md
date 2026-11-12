# halo-deploy

Ansible + Jenkins deployment of Vantix screening products to customer sites.

## Products

| Product | What | `halo_product` |
|---|---|---|
| **Lane** | X-ray lane screening | `lane` |
| **CT** | CT checkpoint screening | `ct` |
| ~~Gatekeeper~~ | **Retired in 2025.** All customers migrated to Lane. Do not deploy, fix or extend it. | — |

Lane and CT share the `halo` role. Product-specific behaviour is gated on `halo_product`,
so a change to the role affects both products unless gated.

## Sites

A site is one backend host (`halo_backends`) plus N station hosts (`halo_stations`).
Site inventories live in `inventories/sites/`.

## Delivery flow

```
component CI -> registry.vantix.example -> kit (versioned bundle: image tags + config files, dev/rc/prd)
            -> SiteDesk (per-site configuration app, owned by the App team) triggers Jenkinsfile_halo
            -> Jenkins renders inventories/jenkins-site.yaml (replaceable-* placeholders <- build params)
            -> provision-halo.yaml
```

**Detector packs (the detection model) are not part of the kit.** Customers need different
detections, so the pack is chosen per site in SiteDesk and passed to the job separately.

Adding a deployment option therefore touches four places: the role default, a placeholder in
the Jenkins template inventory, a Jenkinsfile parameter plus its `sed` substitution, and a
SiteDesk field (App team).

## Offline (USB) upgrades

Most airport stations have no internet access. Field engineers upgrade them from a USB stick
built per site by `tools/build-usb` (not included here): the kit's images and config files,
plus the site's detector pack under `packs/`. `tools/offline-upgrade` loads the images and runs:

```
ansible-playbook provision-halo.yaml -i <site inventory> --tags offline_upgrade \
  -e halo_config_src=file:///media/usb/config-files
```

Consequences for role code:

* Anything a task fetches must come through a `*_src` variable that the offline flow can
  repoint at the USB stick. Add a new one (and wire it into `tools/offline-upgrade`) if needed.
* Tasks the offline flow must run need the `offline_upgrade` tag.

## Rolling back a detector pack update

Packs live in versioned directories under `/opt/halo/packs/`. `current` is a symlink to
the active pack and `previous` is a symlink to the pack that `current` replaced. Old pack
directories are never deleted. To see what a station runs:

```
ls -l /opt/halo/packs
```

To go back to the previous pack on a station, without a USB stick or network:

```
cd /opt/halo/packs
ln -sfn "$(readlink previous)" current
docker restart lane-inference
```

Or re-run the update from the USB stick with the old pack version, which needs the old pack
and its line in `manifest.sha256` on the stick:

```
ansible-playbook provision-halo.yaml -i <site inventory> --tags offline_upgrade \
  -e halo_pack_src=file:///media/usb/packs -e halo_pack_version=<old version>
```

Check afterwards that the station really runs the old pack. A kit rollback only swaps the
images and config, not the pack.

## Conventions

* A second run must report no changes.
* Tag every task.
* No plaintext secrets: Ansible Vault only (`vault_*` variables, not included in this repo).
  Tasks handling secrets get `no_log: true` and `diff: false`.
* Every new variable gets a default in `defaults/main.yaml` that keeps existing sites unchanged.
