---
title: Prepare up your local environment with `pyenv` and `mise-en-place`
date: 2024-03-14 21:29
categories:
  - programming
  - linux
---

I try to keep as much of my relevant toolchain as up to date as possible,
without loosing control over it and breaking compatibility for different
projects.

This boils down to

- Enable independent versions of the same program to be installable
  - e.g. I want java-21 but also java-17 in a different project
- Define the installed/active programs and configuration as code
- Enable per-context activation of different versions

There are many different approaches to achieve this, e.g.

- Virtualization
  - Manually managed Project-context virtual machines
  - Built-in e.g. with [qubes-OS](https://www.qubes-os.org/) or
    [spectrum-os](https://spectrum-os.org/)
  - Powerful due to the level of isolation (security, flexibility)
  - Highest overhead
- Containerization
  - [devcontainers](https://containers.dev/)
  - Custom build containers
- Alternative distributions/package managers
  - [NixOS](https://nixos.org/)
  - [Guix](https://guix.gnu.org/)
  - Fully declarative environment
- shim managers
  - [pyenv](https://github.com/pyenv/pyenv)
  - [nodeenv](https://github.com/ekalinin/nodeenv)
  - [asdf](https://fig.io/manual/asdf/shim)
  - [mise-en-place](https://mise.jdx.dev/)

I have played around/worked with a few of the given options and changed my
setup a few times already. Maybe the observations might be useful to you, or my
later self.

## Boundary conditions

The boundary conditions of my setup are as follows

- The target system is Linux only. My windows experience is not too deep, but I
  don't think that something I want is easily achievable on windows.
- I have most of my system setup experience with Debian, as I have been
  running, breaking, fixing and working with Debian unstable for 17 years.
- I have quite a bit of experience with Ubuntu
- In general, I am very happy with Debian and am not interested in
  distro-hopping. Switching away from Debian would have to be motivated by
  significant advantages and no downsides
- I want my setup to be portable between my personal (Debian based) machines
  and machines I don't have full control over

## Path to the current setup

### Nix as package manager

Nix, and with it NixOS, revolve around a functional domain specific language
that can fully declare environments, including installed packages/tools, making
use of the `Nixpkgs` packages collection, as well as their configuration.

These environments are fully self-contained, reproducible and very flexible.
With Nix, it is possible to declare the desired _system_ state, but also
independent environments in e.g. the context of a specific development project,
or a temporary test environment that can be created and discarded afterwards.

This example defines a `nix-shell` environment with a specific ruby version:

```text
{ pkgs ? import <nixpkgs> {} }:
  pkgs.mkShell {
    # nativeBuildInputs is usually what you want -- tools you need to run
    nativeBuildInputs = with pkgs.buildPackages; [ ruby_3_2 ];
}
```
