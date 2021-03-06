How to set up your own Hydra Server
===================================

This repository contains a complex'ish example configuration for the Nix-based
continuous build system [Hydra](http://nixos.org/hydra/) that new users can use
to get started. The file [`hydra-common.nix`](hydra-common.nix) defines basic
properties of a VBox-based virtual machine running NixOS 15.09, which
[`hydra-master.nix`](hydra-master.nix) extends to configure a running Hydra
server. [`hydra-slave.nix`](hydra-slave.nix), on the other hand, configures a
simple build slave for the main server to delegate build jobs to. Finally,
[`hydra-network.nix`](hydra-network.nix) ties those modules together into a
network definition for Nixops.

To run these examples quickly with `nixops` on your local machine, you'll need

- hardware virtualization support,
- 8+ GB of memory,
- [NixOS](http://nixos.org/) and [Nixops](http://nixos.org/nixops/) installed.

Also, your `configuration.nix` file should define:

~~~~~ nix
virtualisation.virtualbox.host.enable = true;
~~~~~

If those pre-conditions are met, follow these steps:

1. Generate an SSH key used by the Hydra master server to authenticate itself
    to the build slaves:

    ~~~~~ bash
    $ ssh-keygen -C "hydra@hydra.example.org" -N "" -f id_buildfarm
    ~~~~~

2. Set up your shell environment to use the `nixos-15.09` release for all
    further commands:

    ~~~~~ bash
    $ NIX_PATH="nixpkgs=https://github.com/nixos/nixpkgs-channels/archive/nixos-15.09.tar.gz"
    $ export NIX_PATH
    ~~~~~

3. Make sure that your system knows the contents of the binary cache for that
    branch! If your system is subscribed to the `release-15.09` channel, then
    everything is fine. If you're not -- i.e. because you're running on
    `unstable` --, then you'll have to add a subscription by running (as
    `root`) the command:

    ~~~~~ bash
    $ nix-channel --add https://nixos.org/channels/nixos-15.09 nixos-15.09
    $ nix-channel --update
    ~~~~~

    Futhermore, expend your `$NIX_PATH` to include the channel
    information:

    ~~~~~ bash
    $ NIX_PATH+=":/nix/var/nix/profiles/per-user/root/channels/nixos"
    ~~~~~

4. Start the server:

    ~~~~~ bash
    $ nixops create -d hydra hydra-network.nix
    $ nixops deploy -d hydra
    ~~~~~

5. Ensure that the main server knows the binary cache for `nixos-15.09`:

    ~~~~~ bash
    nixops ssh hydra -- nix-channel --update
    ~~~~~

If all these steps completed without errors, then `nixops info` will tell you
the IP address of the new machine(s). For example, let's say that the `hydra`
machine got assigned the address `192.168.56.101`. Then go to
`http://192.168.56.101:8080/` to access the web front-end and sign in with the
username "`alice`" and password "`foobar`".

Now you are ready to create projects and jobsets the repository contains the
following examples that you can use:

- [`jobset-nixpkgs.nix`](jobset-nixpkgs.nix)
- [`jobset-libfastcgi.nix`](jobset-libfastcgi.nix)
- [`jobset-jailbreak-cabal.nix`](jobset-jailbreak-cabal.nix)

The last jobset performs several Haskell builds that may be quite expensive, so
it's probably wise *not* to run that on virtual hardware but only on a real
sever.

Miscellaneous topics
--------------------

- How to [disable binary substitutions](https://github.com/NixOS/hydra/commit/82504fe01084f432443c121614532d29c781082a)
   for higher evaluation performance.


- How to run emergency garbage collections:

    ~~~~~ bash
    $ systemctl start hydra-update-gc-roots.service
    $ systemctl start nix-gc.service
    ~~~~~

- "Shares" are interpreted as follows: each jobset has a "fraction", which is
   its number of shares divided by the total number of shares. The queue runner
   records how much time a jobset has used over the last day as a fraction of
   the total time and then jobsets are ordered by their allocated fraction
   divided by the fraction of time used i.e. jobsets that have used less of
   their allotment are prioritized.
