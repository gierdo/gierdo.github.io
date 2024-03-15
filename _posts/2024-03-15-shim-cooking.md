---
title: Cook your local environment with `mise-en-place`. And others.
date: 2024-03-15 21:29
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
- Alternative distributions/package managers
  - [NixOS](https://nixos.org/)
  - [Guix](https://guix.gnu.org/)
  - Fully declarative environment
- Containerization
  - [devcontainers](https://containers.dev/)
  - Custom build containers
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

### Virtualization

On my personal setup, I don't have any usecases requiring windows or other
non-Linux operating systems, and my security requirements are not _that_ high.
Consequently, I have not created and worked with a virtualization setup enough
to form an opinion strong enough to write about.

### Nix

#### Basics

Nix, and with it NixOS, revolve around a functional domain specific language
that can fully declare environments, including installed packages/tools, making
use of the `Nixpkgs` packages collection, as well as their configuration.

These environments are fully self-contained, reproducible and very flexible.
With Nix, it is possible to declare the desired _system_ state, but also
independent environments in e.g. the context of a specific development project,
or a temporary test environment that can be created and discarded afterwards.

This example defines a `nix-shell` environment with a specific ruby version:

```nix
{ pkgs ? import <nixpkgs> {} }:
  pkgs.mkShell {
    # nativeBuildInputs is usually what you want -- tools you need to run
    nativeBuildInputs = with pkgs.buildPackages; [ ruby_3_2 ];
}
```

What's especially powerful with Nix is the fully self-contained and
reproducible build definition, which includes the full definition of the build
environment, as shown in this example, defining the build of `chord`:

```nix
{
  pkgs ? import (fetchTarball {
    url = "https://github.com/NixOS/nixpkgs/archive/4fe8d07066f6ea82cda2b0c9ae7aee59b2d241b3.tar.gz";
    sha256 = "sha256:06jzngg5jm1f81sc4xfskvvgjy5bblz51xpl788mnps1wrkykfhp";
  }) {}
}:
pkgs.stdenv.mkDerivation rec {
  pname = "chord";
  version = "0.1.0";

  src = pkgs.fetchgit {
    url = "https://gitlab.inria.fr/nix-tutorial/chord-tuto-nix-2022";
    rev = "069d2a5bfa4c4024063c25551d5201aeaf921cb3";
    sha256 = "sha256-MlqJOoMSRuYeG+jl8DFgcNnpEyeRgDCK2JlN9pOqBWA=";
  };

  buildInputs = [
    pkgs.simgrid
    pkgs.boost
    pkgs.cmake
  ];

  configurePhase = ''
    cmake .
  '';

  buildPhase = ''
    make
  '';

  installPhase = ''
    mkdir -p $out/bin
    mv chord $out/bin
  '';
```

Nix' development is driven by a very active expert community and happens at an
impressively quick pace. Most activity, understandably, targets NixOS, not Nix
as third-party package manager.

Nix works by having a _package store_, which contains every package in every
installed version in a unique directory, distinguished with a unique hash. In
any requested environment, specific versions of everything are "activated"
specifically.

The result resembles _Containers_ a lot, but with full system integration as
upside and less flexibility as downside. It's even possible to build an [OCI
compatible container
image](https://nix.dev/tutorials/nixos/building-and-running-docker-images.html)
out of a Nix environment.

#### My Caveat

I only tested Nix as package manager on Debian, which, I have to admit, is
somewhat limited. The main real issue I've had was with regards to `OpenGL`,
which is not supported in non-NixOS environments. This prevents any programs to
make proper use of graphics cards, and when I tested it, I tried to define my
`sway` setup with Nix. Which did not work without a lot of hassle.

There [are solutions](https://github.com/nix-community/nixGL), but it's all
pretty meh. Besides that, the package management with Nix was pretty slow,
taking a lot of additional space and (for my usecases and experience) less
powerful than using "normal" OCI containers with Docker/podman.

### Guix

#### Basics

Guix resembles Nix quite a bit. The principle is the same, you use a functional
programming language to fully declare the desired state of your
system/package/..., different versions of stuff are put into a _package store_
thing and are activated specifically, it's a dedicated Distribution (_Guix
System_) and a package manager (_Guix_), etc.

The main difference is that the theoretical foundation is very nice, clean and
impressive. There is an extremely strong focus on _software freedom_, which is
nice, and it doesn't use a domain specific (and hence a bit limited)
description language, but a full fledged functional programming language, _GNU
Guile_, which is a _Scheme_ implementation. It's pretty nice. And also, pretty
weird.

This is an example of a package definition for `postgresql-autodoc`, which I
created and put into my `guix-channel`, which is pretty easy to do. Which is,
again, pretty nice.

```scheme
(define-module (postgresql-autodoc)
               #:use-module (guix utils)
               #:use-module ((guix licenses) #:prefix license:)
               #:use-module (guix packages)
               #:use-module (guix git-download)
               #:use-module (guix build-system gnu)
               #:use-module (gnu packages base)
               #:use-module (gnu packages perl)
               #:use-module (gnu packages web)
               #:use-module (gnu packages databases))

(define-public postgresql-autodoc
               (package
                 (name "postgresql-autodoc")
                 (version "80a6b150febb5c0c91f2daa433cc089ff1278841")
                 (source
                   (origin
                     (method git-fetch)
                     (uri (git-reference
                            (url "https://github.com/cbbrowne/autodoc")
                            (commit version)))
                     (file-name (git-file-name name version))
                     (sha256
                       (base32
                         "0ip1xy5qmm5hj392vxfs5fkgxgdv8d9kj7pp6p4x79npxczkxyzd"))))
                 (build-system gnu-build-system)
                 (arguments
                   '(#:phases
                     (modify-phases %standard-phases
                                    (delete 'configure)
                                    (delete 'check)
                                    (add-before 'install 'patch-prefix
                                                (lambda _
                                                  (substitute* (cons* "Makefile")
                                                               (("/usr/local") (assoc-ref %outputs "out")))
                                                  #t))
                                    (add-after 'install 'wrap-program
                                               (lambda* (#:key outputs #:allow-other-keys)
                                                        (let* ((out (assoc-ref outputs "out"))
                                                               (path (getenv "PERL5LIB")))
                                                          (wrap-program (string-append out "/bin/postgresql_autodoc")
                                                                        `("PERL5LIB" ":" prefix
                                                                          (,(string-append out "/lib/perl5/site_perl"
                                                                                           ":"
                                                                                           path)))))
                                                        #t))
                                    )))
                 (native-inputs
                   `(("which", which)))
                 (inputs
                   `(("perl" ,perl)
                     ("perl-html-template", perl-html-template)
                     ("perl-term-readkey", perl-term-readkey)
                     ("perl-dbd-pg", perl-dbd-pg)))
                 (home-page "https://github.com/cbbrowne/autodoc")
                 (synopsis "PostgreSQL Autodoc")
                 (description
                   "This is a utility which will run through PostgreSQL system tables and returns HTML, DOT, and several styles of XML which describe the database.")
                 (license license:bsd-3)))
```

#### My caveat

Guix doesn't have Nix' problems with _OpenGL_, and it immediately felt much
nicer than Nix to me. I have used it for around a year to manage most of my
environment. The main beef I had with it was details and weird sluggishness in
details. Like with Nix, package management is/can be quite slow. Due to the
nature of things, depending on the current state of the system, it can happen
that you won't install binary packages but build everything from scratch. Which
happens reliably, reproducibly and nicely, as expected. That's the point, after
all. But it still takes SO MUCH TIME!! and the fan noise, and ... Additionally,
for the graphical environment, I had weird issues I couldn't quite resolve.
E.g. starting my desktop (`sway`) would take several seconds longer than with
pure Debian. Similarly, the initial start of my switcher application, `rofi`,
would take something between 3 and 10 seconds, compared to practically instant
on Debian. I spent a few evenings trying to solve the issue, eventually giving
up.

After a while, I accepted, again, that "normal" containers fit my usecase
better.

### Containers

#### Basics

OCI containers work with _namespaces_ to isolate system resources (processes,
networking, users, mountpoints, ipc, time) and virtual filesystems. The desired
state of a container is freezed in a _container image_, which basically defines
the virtual base filesystem in _layers_ and certain system interfaces, from
which an actual _container_ can be started. These _images_ are usually defined
in a `Dockerfile`, which is the build definition of a container image, and can
be stored, distributed and shared. This way, containers can solve problems in
system dependency management, software distribution and orchestration.

This is an example of a `Dockerfile`, defining an environment based on
`ubuntu:jammy`:

```Dockerfile
FROM ubuntu:jammy

RUN apt-get update && apt-get install -y --no-install-recommends \
  ca-certificates \
  wget \
  gpg \
  pkg-config \
  && rm -rf /var/lib/{apt,dpkg,cache,log}/
```

Containers can be built `FROM scratch` or based on existing environments, e.g.
benefiting on a strong community and ecosystem. There are base images from
basically every major Linux distribution and _distroless_ images, limiting the
attack surface of application environments. Base images and final images are
cryptographically identified and can be referenced reproducibly and explicitly.

#### My caveat

I have been using [podman](https://podman.io/) as highlevel container
management solution and alternative to `Docker`, mainly due to it's
(root-)daemonless nature.
I use it to run and expose permanent network based local services (see also
[this]({% link _posts/2023-11-24-llama-vim.md %}) or development environments.

One advantage with `podman` is that even without additional magic, due to the
user ID mapping, the root UID within a container maps to the UID of the user
who started the container outside of the container. Rootless containers for the
win! This allows you to simply mount a workspace or other resources into a
container and work with it inside of the container without having to worry
about file permissions and ownership.

E.g. like this.

```bash
$ podman run --annotation run.oci.keep_original_groups=1 -v $(pwd)/.local/lib/llama/models:/models --device /dev/kfd --device /dev/dri --rm -it localhost/llama-cpp-python-server-rocm:0.2.20 bash
```

I like the flexibility, manageability and reusability given by containers a
lot and use them heavily.

For the creation of `Dockerfiles`, I make use of
[`dockerfile-language-server`](https://github.com/rcjsuen/dockerfile-language-server)
through [`coc-docker`](https://github.com/josa42/coc-docker) and a fancy shell
setup with [`zsh
podman`](https://github.com/ohmyzsh/ohmyzsh/blob/master/plugins/podman/README.md),
and [`lazydocker`](https://github.com/jesseduffield/lazydocker), which makes
writing Dockerfiles and working with containers pretty convenient.

However, system integration is not ideal. Some (most?) of that is by design, a
container is supposed to be separated from the host system, after all. What if
I want to just want to isolate a specific tool and specify it in the context of
a project?

### Shim managers

As the [README](https://github.com/asdf-vm/asdf) of `asdf` states:

> Once upon a time there was a programming language
> There were many versions of it
> So people wrote a version manager for it
> To switch between versions for projects
> Different, old, new.
>
> Then there came more programming languages
> So there came more version managers
> And many commands for them
>
> I installed a lot of them
> I learnt a lot of commands
>
> Then I said, just one more version manager
> Which I will write instead
>
> So, there came another version manager
> asdf version manager - https://github.com/asdf-vm/asdf
>
> A version manager so extendable
> for which anyone can create a plugin
> To support their favourite language
> No more installing more version managers
> Or learning more commands

#### Basics

In come _shim managers_! These programs manage the installation of _other_
programs, capable of updating them, installing and activating different
versions of the desired program and (ideally) integrating seamlessly with the
rest of the environment and ecosystem.

There are _language specific_ managers, e.g. for _python_ there is
[`pyenv`](https://github.com/pyenv/pyenv), which integrates very nicely with
e.g. `poetry` and general python virtual environments. _Rust_ has it's own
installer and toolchain manager, [`rustup`](https://rustup.rs/), which is also
capable of managing different versions of _rust_.

Even more powerful are general purpose shim managers like
[`asdf`](https://fig.io/manual/asdf/shim) and, written in rust, more modern and
compatible with `asdf`, [`mise-en-place`](https://mise.jdx.dev/).

`mise` has a rich ecosystem of plugins, which allows it to manage versions of a
plethora of programs. These can be activated on a system level, temporary shell
level or in the context of (project) directories.

This could be a system level configuration at `~/.config/mise/config.toml`, or
a specific project configuration in `~/workspace/foo/.mise.toml`

```toml
[tools]
go = "1.22.0"
node = "21.6.1"
java = "adoptopenjdk-21.0.2+13.0.LTS"
ruby = "3.3.0"
awscli = "2.15.19"
direnv = "2.34.0"
```

`mise` makes sure that, in the respective context, the specified version of the
specified tools are configured and take precedence in the `$PATH`.

#### My caveat

I have been using `pyenv`, `asdf` and `rustup` for a long time, recently
replacing `asdf` with `mise`. Obviously, `pyenv` and `rustup` are only used to
manage `python` and `rust`.

I have replaced `asdf` with `mise`, as it's much faster, more actively
maintained and maintainable at all. While `asdf` is very impressive, it's
written in `bash`(!), and I'm almost sure that the authors regularly curse
their choice. `mise` is compatible with `asdf`, but extends its functionality.
And it works great!

```bash
$ java --version
openjdk 21.0.2 2024-01-16 LTS
OpenJDK Runtime Environment Temurin-21.0.2+13 (build 21.0.2+13-LTS)
OpenJDK 64-Bit Server VM Temurin-21.0.2+13 (build 21.0.2+13-LTS, mixed mode, sharing)
$ mise shell java@corretto-21.0.2.14.1
mise java@corretto-21.0.2.14.1 downloading amazon-corretto-21.0.2.1 72.68 MiB/199.70 MiB (14s) ███████░░░░░░░░░░░░░  7s
.
.
.
$ java --version
openjdk 21.0.2 2024-01-16 LTS
OpenJDK Runtime Environment Corretto-21.0.2.14.1 (build 21.0.2+14-LTS)
OpenJDK 64-Bit Server VM Corretto-21.0.2.14.1 (build 21.0.2+14-LTS, mixed mode, sharing)
$ mise use java@corretto-21.0.2.14.1 -p ./
mise ./.mise.toml tools: java@corretto-21.0.2.14.1
$ cat .mise.toml
[tools]
java = "corretto-21.0.2.14.1"
```

Integration is absolutely seamless, once it's set up correctly.

If you are interested in the setup, check the (linked) documentation, or take a
look at my [dotfiles](https://github.com/gierdo/dotfiles) :)

## tl;dr

- It's good to have different versions of programs
- It's hard to manage different versions of programs
- Different ways of managing this have different (dis)advantages
- Containers are not too bad sometimes
- Shim managers are also not too bad sometimes
