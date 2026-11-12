# DECISIONS:

## Assessment of Riya's branch
The handover says "ready to merge". It is not. The design is sound; the implementation can't work at Northport.

**Keep:** versioned pack dirs plus a `current` symlink; empty `halo_pack_version` = do nothing; the `PACK_VERSION` parameter; the lab inventory.

**Fixed** (5 commits on `feature/offline-pack-update-fix`):
- Pack and manifest came from hardcoded `https://` URLs, so nothing was read from the USB. Now `halo_pack_src`.
- Pack tasks lacked the `offline_upgrade` tag, so the USB run skipped them.
- `current` was deleted before download and checksum, so a failure left no pack. Now verify first, then switch; old packs stay on disk.
- New `previous` link records the pack that was replaced, so a station can be switched back.
- README: rollback steps.


## Friday: ships and doesn't
Ships:

Ships Friday

- VTX-401 (Omer): review the Riya branch and the 4 fixes that must be done by Friday. These are the commits already made on feature/offline-pack-update-fix.
- VTX-415 (Lior or Omer): don't deploy yet. Email Maya (the team lead) about this customer and about the Friday deployment, so she can confirm the priority here and decide whether it is more urgent. Also check the SLA we agreed with the customer before deciding anything. Gatekeeper is retired, so first confirm which product runs in Lisbon.

P0
- VTX-450 (new) (Lior): Jenkins never replaces the replaceable-pack-version placeholder because the sed line is missing. So every job run sets halo_pack_version to that literal string and tries to install a pack with that name on every station.

Doesn't Ships Friday
P2
- VTX-412: make it depend on VTX-450, so SiteDesk doesn't start sending PACK_VERSION before the Jenkins fix is in.
- VTX-423, VTX-425: postpone until after Friday.
- ...
- ...

## Rollout
**Rollout:** the stick holds the kit, `dp-2.3` and `dp-2.1`, and `manifest.sha256` with both. The engineer runs `tools/offline-upgrade` with `-e halo_pack_src=file:///media/usb/packs -e halo_pack_version=dp-2.3`. One station first; confirm it runs dp-2.3 and scans; then the rest.

## Rollback, no internet:**
- On the station, without the stick: `cd /opt/halo/packs && ln -sfn "$(readlink previous)" current && docker restart lane-inference`.
- Or re-run the update from the stick with `halo_pack_version=dp-2.1`.
- Re-running the previous kit does not restore a pack: it swaps images, not packs.

## Assumptions and questions
- `tools/offline-upgrade` and `tools/build-usb` aren't in the repo; I assume they can pass `halo_pack_src` and carry both packs. (Omer)
- Kit 4.9's image supports pack loader v2. (Noa)
- Northport's `current` is already a symlink; if it is a real directory, the update fails and leaves it untouched. (Omer)
- Restart vs. recreate of the container must be tested on the lab station before Thursday. (Omer)
