# Stackable Wrappers (Proposal 634)

- Ticket: https://forums.whonix.org/t/write-draft-for-stackable-wrappers-on-debian-devel/18776
- File: https://github.com/Kicksecure/proposals/blob/master/634-stackable-wrappers.md

# Description

In Debian, there is no such thing as stackable wrappers. It would be desirable to have them.

A wrapper in this context is defined as a minimal script that automatically prepends something in front of a program the user wants to run or appends something, such as default command-line parameters.

Here are some examples of commands to prepend:

* `firejail firefox`
* `torsocks gpg`
* `LD_PRELOAD="$LD_PRELOAD":libeatmydata.so rsync`
* `bindp`, `timeprivacy`, and probably a lot more
* Likely quite a few `dpkg` diversions are used for this purpose

Here are some examples of commands to append:

* For ZeroNet, it would be useful to always automatically append, for example, `--tor always`.

One can have one wrapper using `dpkg-diversions` / symlinks, but it becomes harder to stack these wrappers, especially when the wrappers are supposed to be installed by different packages.

Ideally, there would be something like an `/etc/wrapper.d` folder, where packages and users could drop wrappers that get stacked in lexical order. For example:

* `/etc/wrapper.d/30_usr.bin.gpg_torsocks.conf`
* `/etc/wrapper.d/40_usr.bin.gpg_firejail.conf`
* The first script, `30_usr.bin.gpg_torsocks.conf`, would prepend `torsocks`, and the second script, `40_usr.bin.gpg_firejail.conf`, would prepend `firejail`.
* How the wrappers would be specified in the config language is yet to be invented, which will be done if this implementation path looks favorable.

## Our Current (Insufficient) Options

### dpkg Diversions / Symlinks
* Only one `dpkg-diversion` per command is allowed.
* When using a `dpkg-diversion`, for example, for `curl` to prepend `torsocks`, one can no longer use `killall curl` but must use `killall curl.dpkg-diversion-extension`.
* These can break various things such as AppArmor profiles.
* Other weird things can happen. For example:
  * `~/.local/share/Ricochet/ricochet/ricochet.json` becomes
  * `~/.local/share/Ricochet/ricochet.anondist-orig/ricochet.json`.

### .desktop Files
* `.desktop` files are not a solution, as they do not work for applications started from a terminal emulator or virtual terminal.

### /usr/local
* Not allowed for packages.
* `firejail` currently uses `/usr/local`, but the user has to manually run `firecfg`, because automatically running it would violate Debian policy, as writing to `/usr/local` is not allowed for packages.

### Leaving It to the User
* Beginner users cannot be expected to start a terminal every time they want to launch a program. This usability problem can hinder widespread adoption.

### Amend PATH Environment Variable by User
* Not beginner-friendly.
* Does not work for services started by `init` / `systemd`.

### Amend PATH Environment Variable by Package
* `firejail` wrappers will be installed to the `/usr/lib/firejail/` folder, for example, `/usr/lib/firejail/firefox`.
* Invent a drop-in folder to amend the `PATH` variable, like `/etc/path.d`. For example:
  * `/etc/path.d/40_firejail.conf`:
    ```
    PATH="$PATH":/usr/lib/firejail/
    ```
  * `/etc/path.d/50_debian_default.conf`:
    ```
    PATH="$PATH":/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
    ```
  * This would result in the `PATH` being set to:
    ```
    /usr/lib/firejail/:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
    ```
* By calling, for example, `firefox`, first the `firejail` wrapper `/usr/lib/firejail/firefox` would run, remove itself (`/usr/lib/firejail/`) from `PATH`, prepend `firejail`, and run `firefox`. Perhaps another user-specific wrapper in `/usr/local/bin/firefox` could run that finally executes the real `firefox` program in `/usr/bin/firefox`.

### Real-World Example
Whonix would like to prepend both `torsocks` (for stream isolation) and `firejail` (for containment) in front of `gpg`.

## Related
* `firejail` - Feature Request: Automatically starting programs under `firejail` - https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=822693

What do you think? How could this be implemented? What would be the best place to implement this?

Feedback on the ticket [0] and pull requests on the repository [1] are welcome.

[0]: https://forums.whonix.org/t/write-draft-for-stackable-wrappers-on-debian-devel/18776
[1]: https://github.com/Kicksecure/proposals/blob/master/634-stackable-wrappers.md
