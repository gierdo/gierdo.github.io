---
title: Secure(ish) environment setup - direnv + gnome-keyring
categories:
  - programming
  - linux
tags:
  - security
  - direnv
  - environment
---

# How and why to set up automated environment secret stuff

## The Problem

When working on several projects in different environments, interacting with
distributed and relatively loosely coupled _systems_ and _platforms_, such as
different _Jira_, _GitLab_, _..._ instances and other _API_ providers, e.g.
_OpenAI_ or an _Artifactory_ or whatever.

I want to be able to _consume_ these systems from my development environment,
which is basically my shell with dedicated _command line interfaces_ and
_integrations_ in `vim`.

How can I now configure all the endpoints and other _environmental aspects_,
including _API tokens_ and other _secrets_, independently for different project
contexts, without having to copy-paste secrets or hard-code secrets somewhere,
e.g. putting clear-text tokens into `.env` files?

The solution should handle

- Automation and convenience. Whenever I enter a specific directory that
  specifies a context, e.g. a project or group of projects, I want the
  environment to automagically be set up correctly. When I leave the
  environment or change context, I want the environment to be teared down
  again.
- Centralization. _Don't repeat yourself_ (DRY) is not just a good practice for
  programming. I don't want to have 27 places to touch when I changed my GitHub
  API token.
- Security. _No_ clear-text secrets stored somewhere, ever.

## Possible Solution

In pretty much every Linux environment that is not KDE, you should have the
`gnome-keyring` available. The `gnome-keyring` is basically a local database
that is _encrypted_ with the user's login password (-> automatically decrypted
on login through pam), which provides a convenient centralised interface for
secrets/keys. `gnome-keyring` supports gpg and ssh keys, which it can unlock
and act as a key agent for, and generic passwords.

All passwords can then be retrieved by libsecret and the according tooling.
There is a built-in git integration (built into git, but you still have to
configure it!), and many applications use it already, e.g. the network manager,
Chrome, Chromium, Edge, ...

This opens a solution path, if it would be possible to automatically _retrieve_
secrets from the `gnome-keyring` and create _environment variables_ etc.

## Solution

There is a _cli_, `secret-tool`, which allows you to store and retrieve generic
secrets from/into the `gnome-keyring`. In combination with your shell setup
through `direnv`, this allows for pretty nice and still secure automation,
fulfilling all the goals.

### Setup

#### gnome-keyring

Well. You need the `gnome-keyring` to be set up, ideally unlocked automatically
on login. If you're on gnome, that should come as a given. If you are using a
different environment, it might be a given.

#### secret-tool

Install the `secret-tool`. On _Debian_ based systems, which includes all the
_Ubuntu_ flavours, that's done with

```bash
sudo apt-get install libsecret-tools
```

#### direnv

Install `direnv`, e.g., if you are on _Debian_ based systems, with

```bash
sudo apt-get install direnv
```

And make sure to set it up and integrate it into your shell.
I am using zsh, so you will find the configuration in the [`.zshrc`](https://github.com/gierdo/dotfiles/blob/c8877582a3f8fce10e5c78f4e9dc70b8e6a6e9fc/.zshrc#L148)

```zsh
...
if command -v direnv 1>/dev/null 2>&1; then
  _evalcache direnv hook zsh
fi
...
```

### How does it work?

For example, I want to have a gitlab API token in all my repositories for
<https://private-gitlab.com>, so that I can quickly create new projects or open
/ interact with merge requests with the _gitlab cli_, `glab`. So, I have a file
`~/workspace/private-gitlab.com/.envrc` with the following content:

```bash
export GITLAB_TOKEN=$(secret-tool lookup protocol https server private-gitlab.com user <bob@secret.com>)
export GITLAB_URI="<https://private-gitlab.com>"
export GITHUB_TOKEN=$(secret-tool lookup protocol https server github.com user <dominik.gierlach@gmail.com>)
```

Together with `direnv` set up and integrated in to the `shell`, as can be seen
im my [dotfiles.](https://github.com/gierdo/dotfiles), this retrieves a token
from the keyring whenever I enter a repository in
`~/workspace/private-gitlab.com/...` and sets it up as local environment
variable in this context.

### Add / change secrets

The secrets themselves can be set up like this:

```bash
secret-tool store --label='Git: <https://github.com/>' schema org.gnome.keyring.NetworkPassword protocol https user <dominik.gierlach@gmail.com> server github.com
```

Or by using `seahorse`, a graphical manager for the `gnome-keyring`.

### Nested environments

With `direnv`, it's possible to nest environments in order to extend them.

Assume a directory structure like this

```text
~/workspace/github.com/..
  - .envrc
+ - project_group_1
  + - project_a
  + - project_b
+ - project_group_2
    - .envrc
  + - project_c
  + - project_d
```

I want to be able to use the environment configuration that is part of
`~/workspace/github.com/.envrc` in every project, but I want to change/extend it
for all projects in `~/workspace/github.com/project_group_1`.

That is possible by referring to the overarching `.envrc` from
`~/workspace/github.com/project_group_1/.envrc`:

```bash
source ~/workspace/github.com/.envrc
export AWS_PROFILE=PROJECT_GROUP_2_AWS
```

## Bonus: Setup git credential integration with `gnome-keyring`

As written above, `gnome-keyring` can also store `git` credentials, when using
_https_ authentication. While this is somewhat orthogonal to setting up shell
environments with secrets, it's connected and important.

`git` can use [_credential helpers_](https://git-scm.com/docs/gitcredentials)
to store and automagically use credentials for different repositories. The
trivial and built-in _credential helper_ is `store`, which simply stores the
credentials on disk in _clear text_, so let's not go down that path.

The _credential helper_ to integrate with `gnome-keyring` on linux is
`git-credential-libsecret`.

### Install `git-credential-libsecret`

On Debian based systems, including Ubuntu, the credential helper is not
available as compiled binary, but the sourcecode is part of the `git` package.

Install it e.g. like this:

```bash
sudo apt-get install libsecret-1-dev
mkdir -p ~/workspace/other/libsecret
cd ~/workspace/other/libsecret
cp -r /usr/share/doc/git/contrib/credential/libsecret/* ./
make
mkdir -p ~/.local/bin
cp git-credential-libsecret ~/.local/bin
```

### Set up `git-credential-libsecret`

After it has been installed, let's make git use it.

```bash
git config --global credential.helper libsecret
```

### Automatic setup with dotfiles and tuning

Yes, that was annoyingly complex, especially for a thing that should not be
forgotten whenever a new machine is set up.

That's why I integrated the setup into my
[dotfiles.](https://github.com/gierdo/dotfiles)
I don't want to have to remember (= look up and re-understand) how to set it up
whenever I set up a new machine, and I don't want to forget to set it up at
all.

The git configuration is part of the [global
`.gitconfig`](https://github.com/gierdo/dotfiles/blob/c8877582a3f8fce10e5c78f4e9dc70b8e6a6e9fc/.gitconfig#L31),
installation is automated through
[tuning](https://github.com/gierdo/dotfiles/blob/c8877582a3f8fce10e5c78f4e9dc70b8e6a6e9fc/tuning/programs.toml#L175).

## tl;dr

- Sometimes secrets are needed in (shell based) environments, to be set up as
  environment variables
- Tokens can be stored centrally and encrypted in the `gnome-keyring`
- Tokens can be retrieved and exported as environment variables with
  `secret-tool`
- Environment specific automation thereof can be done with `direnv`
