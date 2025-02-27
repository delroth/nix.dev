---
myst:
  html_meta:
    "keywords": "tutorial, declarative, shell, environment, developer, nix, nixpkgs"
---

(declarative-reproducible-envs)=
# Declarative shell environments with `shell.nix`

## Overview

Declarative shell environments allow you to

- Automatically run bash commands during environment activation
- Automatically set environment variables
- Put the environment definition under version control and reproduce it on other machines

### What will you learn?

In the {ref}`ad-hoc-envs` tutorial, we looked at imperatively creating shell environments using `nix-shell -p`, for when we need a quick way to access some tools without having to install them globally.
We also saw how to execute that command with a specific Nixpkgs revision using a Git commit as an argument, to recreate the same environment used previously.

In this tutorial we'll take a look how to create reproducible shell environments given a declarative configuration in a {term}`Nix file`.

### How long will it take?

30 minutes

### What will you need?

- A basic understanding of the [Nix language](reading-nix-language)

## Entering a shell with Python installed

Suppose we a development environment in which Python 3 was installed.
The simplest possible way to accomplish this is via the `nix-shell -p` command:
```
$ nix-shell -p git neovim nodejs
```

This command works, but there's a number of drawbacks:
- You have to type out `-p git neovim nodejs` every time you enter the shell.
- It doesn't (ergonomically) allow you any further customization of your shell environment.

A better solution is to create our shell environment from a `shell.nix` file.

## A basic `shell.nix` file

Create a file called `shell.nix` with these contents:

```nix
let
  nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-22.11";
  pkgs = import nixpkgs { config = {}; overlays = []; };
in

pkgs.mkShell {
  packages = with pkgs; [
    git
    neovim
    nodejs
  ];
}
```

::::{dropdown} Detailed explanation
We use a version of [Nixpkgs pinned to a release branch](<ref-pinning-nixpkgs>), and explicitly set configuration options and overlays to avoid them being inadvertently overridden by [global configuration](https://nixos.org/manual/nixpkgs/stable/#chap-packageconfig).

`mkShell` is a function that produces a shell environment.
It takes as argument an attribute set.
Here we give it an attribute `packages` with a list containing one item from the `pkgs` attribute set.

:::{Dropdown} Side note on `packages` and `buildInputs`
You may encounter examples of `mkShell` that add packages to the `buildInputs` or `nativeBuildInputs` attributes instead.

`nix-shell` was originally conceived as a way to construct a shell environment containing the tools needed to debug package builds.
Only later it became widely used as a general way to make temporary environments for other purposes.

`mkShell` is a [wrapper around `mkDerivation`](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-mkShell), so it takes the same arguments as `mkDerivation`, such as `buildInputs` or `nativeBuildInputs`.
The `packages` attribute argument to `mkShell` is simply an alias for `nativeBuildInputs`.
:::
::::

Enter the environment by running `nix-shell` in the same directory as `shell.nix`:

```console
$ nix-shell
[nix-shell]$
```

`nix-shell` by default looks for a file called `shell.nix` in the current directory and builds a shell environment from the Nix expression in this file.
Packages defined in the `packages` attribute will be available in `$PATH`.

Check that the desired packages are indeed available in the expected version as we did in the [previous tutorial](check-package-version).

## Environment variables

You may want to automatically export certain environment variables when you enter a shell environment.


Set your `GIT_EDITOR` to use the `nvim` from the shell environment:

```diff
 let
   nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-22.11";
   pkgs = import nixpkgs { config = {}; overlays = []; };
 in

 pkgs.mkShell {
   packages = with pkgs; [
     git
     neovim
     nodejs
   ];

+  GIT_EDITOR = "${pkgs.neovim}/bin/nvim";
 }
```

Any attribute name passed to `mkShell` that is not reserved otherwise and has a value which can be coerced to a string will end up as an environment variable.

:::{dropdown} Detailed explanation

The newly added attribute `GIT_EDITOR` is set to a string composed of the output store path of the `neovim` derivation and the relative path to the `nvim` executable inside that store path.

See the [Nix language tutorial on derivations for details](derivations).

:::{warning}
Some variables are protected from being set as described above.

For example, the shell prompt format for most shells is set by the `PS1` environment variable, but `nix-shell` already sets this by default, and will ignore a `PS1` attribute set in the argument.

If you really need to override these protected environment variables, use the `shellHook` attribute as described in the next section.
:::

## Startup commands

You may want to run some commands before entering the shell environment.
These commands can be placed in the `shellHook` attribute provided to `mkShell`.

Set `shellHook` to output the current repository status:

```diff
 let
   nixpkgs = fetchTarball "https://github.com/NixOS/nixpkgs/tarball/nixos-22.11";
   pkgs = import nixpkgs { config = {}; overlays = []; };
 in

 pkgs.mkShell {
   packages = with pkgs; [
     git
     neovim
     nodejs
   ];

   GIT_EDITOR = "${pkgs.neovim}/bin/nvim";
+
+  shellHook = ''
+    git status
+  '';
 }
```

## References

- [`mkShell` documentation](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-mkShell)
- Nixpkgs [shell functions and utilities](https://nixos.org/manual/nixpkgs/stable/#ssec-stdenv-functions) documentation
- [`nix-shell` documentation](https://nixos.org/manual/nix/stable/command-ref/nix-shell)

