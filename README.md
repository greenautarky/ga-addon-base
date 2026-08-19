# GreenAutarky add-on base images (armv7)

Two thin, GreenAutarky-owned **armv7 (32-bit ARM)** base images for our Home
Assistant add-ons:

| Image | `FROM` | libc | for |
|---|---|---|---|
| `ghcr.io/greenautarky/ga-addon-base-armv7` | `arm32v7/alpine:3.24` | musl | default — `ga_manager`, `ga_zigbee2mqtt`, `ga_default_addon`, `ga_hmvapp_addon`, the dongle flasher |
| `ghcr.io/greenautarky/ga-addon-debian-base-armv7` | `arm32v7/debian:trixie-slim` | glibc | the Debian-native add-ons — `ga_mosquitto`, `ga_influxdbv1`, and anything needing a `manylinux_armv7l` (glibc) Python wheel that has no musl equivalent |

## Why these exist

The SONOFF iHost fleet is **hardware-locked to armv7** (RV1126/RV1109 Cortex-A7 —
aarch64 is physically impossible). Every Home-Assistant-ecosystem armv7 base
image **froze in late 2025**:

- `ghcr.io/hassio-addons/base` — last armv7 = **18.2.1** (2025-10-16); 19.0.0 dropped it.
- `ghcr.io/home-assistant/armv7-base` — last = **3.22-2025.11.1** (2025-11-21).
- `ghcr.io/hassio-addons/debian-base` — last armv7 = **8.1.4** (2025-10-16).

A frozen base gets **no more security updates**. But **Alpine and Debian
themselves still ship armv7 with full security updates** — Alpine 3.24 to
2028-06-01, Debian trixie/armhf to 2028-08-09, packages built in lockstep with
the 64-bit arches. So we assemble our own thin base **FROM a live distro** and
bump it in **one place** — instead of the toil of patching every add-on's
`build_from` at a dead end, and instead of letting an add-on silently rot on an
EOL base (which is exactly how `ga_manager` sat on EOL Alpine 3.20 for 27 months).

## What's in them

Forked from `hassio-addons/addon-base` v18.2.1 (Alpine) and
`hassio-addons/addon-debian-base` v8.1.4 (Debian) — the last armv7-capable
releases — with three changes:

1. **`FROM` a live distro**: Alpine `3.22.2 → 3.24`, Debian `13.1-slim → trixie-slim`.
2. **apk/apt version pins removed**: the upstream pins are distro-version-specific
   (they don't exist on the newer distro), and unpinned within a pinned distro
   major means every build pulls the **latest security patch** in that major —
   which is the point of owning the base.
3. **Tools bumped to current** (all still ship armv7): s6-overlay `3.2.1.0 →
   3.2.3.2`, bashio `0.17.5 → 0.19.0`, tempio `2024.11.2 → 2026.07.0`.

Everything else — the s6-rc service layout, the `/init` entrypoint contract,
bashio, tempio — is upstream-faithful, so these are drop-in replacements for the
frozen bases.

Proven to build **and** run on real armv7 hardware (2026-08-19):
`ga-addon-base-armv7` → Alpine 3.24 / OpenSSL 3.5.7 (musl); `ga-addon-debian-base-armv7`
→ Debian 13.6 / glibc 2.41 / OpenSSL 3.5.6.

## Use in an add-on

```yaml
# <addon>/build.yaml
build_from:
  armv7: ghcr.io/greenautarky/ga-addon-base-armv7:3.24
```

Pin the **distro-major tag** (`:3.24` / `:trixie`), not `:latest` — so a distro
major bump is a deliberate, tested move, while security patches within the major
arrive automatically on rebuild.

## Maintenance

CI (`.github/workflows/build.yaml`) builds both for armv7 and publishes to GHCR
on every push and **weekly** (so the unpinned security patches are actually
pulled). Bumping a distro major = edit the `base`/`tag` in the workflow matrix
+ the `FROM` in the flavor's `Dockerfile`, then re-verify one build. Estimated
~2–4 person-days/year.

## Credit / license

Assembled from the Home Assistant Community Add-ons base images
(`hassio-addons/addon-base`, `hassio-addons/addon-debian-base`, MIT, © Franck
Nijhof). This fork exists solely to keep armv7 alive on a live distro after
upstream froze it. MIT — see `LICENSE`.
