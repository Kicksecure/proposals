# Listen Port Convention (Proposal 635)

- Ticket: https://forums.whonix.org/t/write-draft-for-local-listener-standard-on-debian-devel/18775
- File: https://github.com/Whonix/proposals/blob/master/635-listen-port-convention.txt

# Description

A convention on listen ports, local or all network interfaces, would be desirable.

At the moment, there seems to be no convention for where server applications are configured to listen by default—on `localhost` vs. all interfaces. It appears that decision is left to the upstream author of the software as well as the packager. Then it is up to the system administrator to decide where the server application should listen. There is no suitable mechanism for derivatives to globally modify this setting.

Usually, applications using Tor ephemeral hidden services, such as `ricochet-im`, `onionshare`, `ZeroNet`, and `unMessage`, listen on `localhost` only.

Whonix is a Debian derivative focused on anonymity, privacy, and security. To oversimplify, it preconfigures Debian with these goals in mind.

Due to Whonix's workstation-gateway split design, applications using Tor ephemeral hidden services need to listen on the workstation's external interface rather than on the workstation's `localhost`.

A global configuration file such as `/etc/ricochet-im.conf` works for system administrators but not for derivatives.

Why is a config folder `/etc/ricochet-im.d` strongly preferred over a config file `/etc/ricochet-im.conf`? When a config file like `/etc/ricochet-im.conf` is owned by one package (`ricochet-im`), it cannot be owned by another package. If another package modifies it using `sed` or similar methods, `dpkg` regards that file as user-modified. This can cause issues—next time that config file is changed by upstream, it triggers an interactive `dpkg` conflict resolution dialog for the user, which is a usability bug (example [2]). Additionally, `sed`-style config modifications are a Debian policy violation.

So far, Whonix has had discussions with `ricochet-im`, `onionshare`, `ZeroNet`, and `unMessage`. These projects are all interested in making their applications compatible with Whonix. However, asking each project to create an `/etc/application-specific.d` folder for Whonix to drop a `/etc/application-specific.d/30_whonix.conf` file that says `listen=10.152.152.10` is inefficient. It duplicates effort and is not particularly desirable for these applications, as they have no other need for `/etc/application-specific.d/`.

Having these applications auto-detect Whonix does not seem like a great solution either. It seems unsafe. If the auto-detection code activates as a false positive, users would be at risk. Since this is Whonix-specific, general solutions reusable by anyone are preferred. At least, that aligns with my interpretation of the *nix philosophy.

## Proposed Convention

* Parse in lexical order:
  * `/usr/lib/listen.d`
  * `/usr/local/etc/listen.d`
  * `/etc/listen.d`
  * Optionally, custom folders
  * Similar to how `systemd` parses these folders. For example, start with parsing `/usr/lib/listen.d/30_default.conf`, followed by `/usr/lib/listen.d/31_other.conf`, followed by `/usr/local/etc/listen.d/30_user.conf`, followed by `/etc/listen.d/30_user.conf`, then `/etc/listen.d/40_user.conf`, and so forth.
* The file used to group the configurations would be `/etc/listen.conf`, with the contents:

```
# lines starting with # are ignored

# predefined
include /usr/lib/listen.d/*.conf
include /usr/local/etc/listen.d/*.conf
include /etc/listen.d/*.conf

# custom configurations may be added here
```

* In theory, any path could be added to this file, even within `$HOME`. However, the usefulness of adding configurations to `$HOME` instead of `/usr/` and `/etc/` is uncertain. It would enable customizations when the user lacks privileges to write outside `$HOME`, but running a server on such a system seems unlikely. Feedback on this is appreciated.

* `.conf` files would contain options like:

```
# lines starting with # are ignored

# global fallback setting for all listeners
listen_ip=127.0.0.1
listen_ip=10.152.152.10
listen_ip=0.0.0.0
listen_ip=UNIX-LISTEN:/path/to/<application-name>.sock
listen_ip=eth0

# web interfaces
listen_ip_web=127.0.0.1
listen_ip_web=10.152.152.10
listen_ip_web=0.0.0.0
listen_ip_web=UNIX-LISTEN:/path/to/<application-name>_web.sock
listen_ip_web=eth0

# listen incoming IP
listen_ip_incoming=127.0.0.1
listen_ip_incoming=10.152.152.10
listen_ip_incoming=0.0.0.0
listen_ip_incoming=UNIX-LISTEN:/path/to/<application-name>_incoming.sock
listen_ip_incoming=eth0

# optional application-specific listen port
listen_port_<application-name>=15000

listen_range_<application-name>=16000-17000
```

### Example Configurations

```
# /etc/listen.d/30_default.conf
listen_ip=0.0.0.0

# /etc/listen.d/50_user.conf
listen_ip=127.0.0.1
```

This configuration would result in listening on `0.0.0.0` as well as on `127.0.0.1`.

To disable listeners from a previous, lower-priority configuration file, use `listen_ip=`. For example:

```
# /etc/listen.d/30_default.conf
listen_ip=0.0.0.0

# /etc/listen.d/50_user.conf
listen_ip=
listen_ip=127.0.0.1
```

This configuration would result in listening on `127.0.0.1` only. This behavior is similar to how `systemd` parses unit files.

To prevent different applications from parsing the configuration differently, and to avoid unexpected results, it would be useful to have a Python library and a command-line tool to query it.

Any questions? Any suggestions? What do you think?

Feedback on the ticket [0] and pull requests on the repository [1] are welcome.

[0]: https://forums.whonix.org/t/write-draft-for-local-listener-standard-on-debian-devel/18775
[1]: https://github.com/Whonix/proposals/blob/master/635-listen-port-convention.txt
[2]: https://www.kicksecure.com/wiki/Configuration_Files#dpkg_interactive_conflict_resolution_dialog
