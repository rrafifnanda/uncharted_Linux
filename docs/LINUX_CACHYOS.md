# Uncharted on Arch / CachyOS: build and run notes

This repository is a personal build of [spacepepperoni's Linux
fork](https://github.com/spacepepperoni/uncharted_Linux) of [SuperMedo's Uncharted PC
port](https://github.com/SuperMedo/uncharted_pc). It exists so that this machine can run a
self-built AppImage that updates itself from here instead of waiting for someone else's Linux port:

- `sync-upstream.yml` merges `Supermedo/uncharted_pc` into this fork every 6 hours, builds the merge
  and only lands it on `main` when the Linux build passes; on a conflict or a failing build it
  opens a pull request and leaves `main` alone.
- `build.yml` builds `main` in a Debian 13 container and publishes a release tagged
  `v<version>-linux.<run>`, which is exactly what the launcher's update check looks for.

## Hardware reality check

These notes were written on the machine this build was made for: CachyOS, AMD Radeon iGPU
(RADV RENOIR, Mesa 26.2), ~3.2 GiB of shared graphics memory and 5.6 GiB of RAM. That is below the
project's stated requirements, so expect low frame rates; Uncharted 1 is the most stable of the
three, Uncharted 3 crashes often.

## Dependencies

```sh
pkexec pacman -S --needed --noconfirm base-devel cmake ninja pkgconf python tk sdl3 \
    vulkan-headers vulkan-icd-loader fmt boost robin-map vulkan-memory-allocator xxhash \
    zydis ffmpeg protobuf nlohmann-json openssl zlib miniupnpc glslang spirv-tools \
    fuse2 squashfs-tools
```

`fuse2` is only needed to run AppImages the usual way; `APPIMAGE_EXTRACT_AND_RUN=1` works without
it. `github-cli` is used for the release workflow, not for the build.

## Build

```sh
git clone --recursive https://github.com/rrafifnanda/uncharted_Linux
cd uncharted_Linux
BB_LTO=OFF BB_JOBS=3 bash build.sh
```

- `BB_JOBS=3`: the GPU library has single translation units that need several gigabytes of RAM;
  three parallel jobs is what fits in 5.6 GiB. A normal desktop can raise this.
- `BB_LTO=OFF`: link-time optimization needs far more memory than this machine has. Leave it on
  elsewhere for a faster binary.

Unit tests and the Vulkan smoke test (neither needs a game dump):

```sh
BB_LTO=OFF BB_JOBS=3 bash build.sh --test        # pad, runtime, file-mods, sema, content
./out/bb-gpu-capabilities --live-resolution      # prints the Vulkan device in use
```

On this machine the smoke test prints
`GPU: AMD Radeon Graphics (RADV RENOIR), integrated or other, 3285 MiB: live resolution changes off`.

## Package and install

```sh
bash packaging/linux/package.sh
# dist/uncharted-linux.tar.gz and dist/Uncharted-x86_64.AppImage
```

An AppImage built on this machine needs glibc 2.44 (Arch/CachyOS current); the release built by CI
on Debian 13 needs only glibc 2.41, which is what SteamOS 3.7 and most desktops have. Use the
locally built one here, the released one elsewhere.

The installer puts the newest release in `~/Applications` with a menu entry and a desktop icon.
`--uninstall` removes those and keeps saves:

```sh
curl -fsSL https://raw.githubusercontent.com/rrafifnanda/uncharted_Linux/main/install.sh | sh
cp dist/Uncharted-x86_64.AppImage ~/Applications/Uncharted-x86_64.AppImage   # local build instead
```

## Running the games

You need your own decrypted, extracted dump of **Uncharted: The Nathan Drake Collection
(CUSA02320) v1.00**; point the launcher's Game tab at that folder. Settings that fit this machine:
720p, FSR and frame generation off, and start with Uncharted 1. Launcher settings live in
`~/.config/uncharted-launcher`, game data (saves, generated files) in
`~/.local/share/uncharted-linux`; the AppImage itself is read-only. To use **Add to Steam**, sign in
to Steam once – the launcher looks for a `userdata` folder and otherwise reports that Steam was not
found (on this machine that folder does not exist yet, Steam has not been signed in to).

## Update flow

1. A push or the scheduled sync lands on `main`; `build.yml` publishes `v0.3.3-linux.<n>`.
2. The launcher reads `LINUX_BUILD` from inside its own AppImage, asks
   `api.github.com/repos/rrafifnanda/uncharted_Linux/releases/latest` and offers that update.
3. To absorb upstream by hand: **Actions → Sync with upstream → Run workflow**.
4. A merge that conflicts or fails its build never reaches `main`; it arrives as a pull request.

## Fixes this copy carries over the upstream fork

All three only matter for local builds (CI does not run the tests and has no spaces in its
checkout path):

| Commit | What it fixes |
| --- | --- |
| `Pad test: link without libbbgpu…` | `tests/test_pad.c` stubs the GPU library's mouse-look entry points (the test could not link at all) and sets `BB_PAD_HOST`, so its SDL virtual controller is read instead of ignored by the scripted pad file. |
| `Packaging: take bundled library paths from ldd as they are` | `packaging/linux/package.sh` no longer splits `ldd` output into fields: Arch prints the loader as `/lib64/ld-linux-… => /usr/lib64/ld-linux-…`, and a checkout path may contain spaces. It also stops with a clear error when a library is missing. |
| `Point the update check, installer and README at this fork` | `RELEASES_API`/`RELEASES_PAGE`, `REPO_URL`, `install.sh` and the README use this repository, so builds from here update themselves from here. The Linux port is still credited to spacepepperoni. |

## Troubleshooting

- **AppImage does not start**: run it from a terminal to see the message;
  `APPIMAGE_EXTRACT_AND_RUN=1 ./Uncharted-x86_64.AppImage` if FUSE is unavailable.
- **`cannot open shared object file`**: the AppImage bundles what distributions may lack, but
  graphics drivers, glibc and libstdc++ come from the system – keep Mesa up to date.
- **`GLIBC_2.44 not found` on another machine**: that is a locally built AppImage; use the release
  (Debian 13) build there.
- **Update check**: it uses the repository baked into the build (`RELEASES_API`); a fresh local
  build with an empty `LINUX_BUILD` file still offers the newest release as an update.
