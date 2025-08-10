# tku137's ZMK Config

![Board](https://img.shields.io/badge/board-Anywhy%20Flake%20S-blue)
![GitHub Actions Workflow Status](https://img.shields.io/github/actions/workflow/status/tku137/zmk-config/build.yml?branch=main)

## Keymaps

### Anywhy Flake S

<img alt="keymap" width="100%" src="./keymap-drawer/anywhy_flake.svg">

## Local development setup

Refer to the [ZMK docs](https://zmk.dev/docs/development/local-toolchain/setup/native) on how to set up a local dev environment for ZMK. This config contains a `.mise.toml` file that sets up everything for you and provides handy mise tasks to trigger build commands, but does not take care of host dependencies for zephyr. Refer to the [Zephyr docs](https://docs.zephyrproject.org/3.5.0/develop/getting_started/index.html#install-dependencies) for host dependencies.

Mise takes care of all the rest and provides tasks to build, update, clean, ...

## Licensing

This repository and any firmware binaries it produces are released under the **GNU GPL v3.0**.  
The combined work includes:

- **ZMK Firmware** (core framework), which is licensed under the **MIT License** (a GPL-compatible, permissive license).
- **flake-zmk-module** (Anywhy Flake shield definitions), which is licensed under the **GNU GPL v3.0**.

Because the flake-zmk-module is GPL-3.0, the final distributed firmware—containing both MIT-licensed and GPL-licensed components—must itself be licensed under **GPL v3.0**, with the full source code and GPL text made available.
