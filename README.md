# Abstract
The aim is to come up with a more convenient way of easily supporting
structured data for external programs, than having to define aliases such as
```nu
alias "ip addr" = (ip --json addr | from json)
```
Note, this also doesn't work very well, as it doesn't put the rest of the
arguments in the right places, e.g. `ip addr show dev eth0` doesn't work.
Commands can work around this, but at the potential cost of tab completion.
```nu
def "nix search" [...args] { ^nix search --json $args | from json }
```
Also note that not every program takes the `--json` argument in the same place,
or even in the same way, e.g. `systemctl -o json`

The primary goal is user friendliness, in particular for interactive use (for
scripting use a bit of verbosity is more acceptable, especially for
portability). To facilitate this, secondary goals are ease of implementation
for commands first (as there are more of them) and shells second (but hopefully
both). Also, having a single way to achieve this would be grand (a tiny library
per OS is acceptable, but I want to avoid having multiple standards or
something too heavy).

# Structured Shell IO
Traditional command-line tends to be line-oriented, but there are some shells
supporting structured data, such as:
- [Powershell](https://learn.microsoft.com/en-us/powershell/)
- [Elvish](https://elv.sh/)
- [nushell](https://nushell.sh/)
- [Xonsh](https://xon.sh/)

See https://github.com/KoviRobi/structured-shell-stdio/issues/1 for cross-references.

On the flipside some programs support returning structured data (often as
JSON). For example:
- [iproute2](https://wiki.linuxfoundation.org/networking/iproute2), e.g.
  `ip -j addr`
- [systemd](https://systemd.io/)
- [nix](https://nixos.org/), e.g. `nix search --json`

See https://github.com/KoviRobi/structured-shell-stdio/issues/2 for cross-references.

But it is a pain to specify an alias for each program to send JSON data. A more
automated way would be handy.

# Two way communication
We need two-way communication, something line JSON for `stdio`. For now I will
concentrate on `stdout`, as often once you have structured data you'll be
manipulating it with shell primitives which understand the structured data.

But we need two-way communication in a different sense too:
1. Tell the called program to output JSON
2. Tell the shell that the output isn't just plain text but actually JSON

# Proposal
One way to achieve it is via having another file-descriptor, just like we have
for `stdout`. We can tell called programs we support JSON by having an
environment-variable, say `JSON_STDOUT=<fd>` where `fd` is a valid
file-descriptor opened by the shell (just like `stdio`). This way the shell
knows that any output on the said file-descriptor

# Security
The program should ensure the file descriptor is valid at the start, before
doing anything else. This is to avoid potential attacks where an attacker will
inject the `JSON_STDOUT` for a shell which doesn't support this, hence won't
set the `JSON_STDOUT` itself. The attacker could then set it to say `3`, which
might be the error log output (say world readable) whereas the `JSON_STDOUT`
might contain privileged data.

# OS Compatibility
- ??? Linux/*BSD/macOS
- ??? Windows

# Streamed data
JSON is unfortunate in that the whole data needs to be output before it can be
parsed. Alternatives to this are [JSON Lines](https://jsonlines.org/) and [JSON
Text Sequences](https://www.rfc-editor.org/rfc/rfc7464). The benefit of JSON
Lines is that it doesn't need the text-editor to support non-printable
characters, whereas the benefit of JSON Text Sequences is that each record
could also be formatted over multiple lines. The latter is probably less
important for shells that already support structured data.

# Alternatives
Message buses such as `dbus` can also communicate with programs. I think the
closest a shell has to good support for `dbus` is [xonsh](https://xon.sh/) when
used with [pydbus](https://pypi.org/project/pydbus/). This might work better
for things where you need to query an existing program, such as `systemd`,
rather than when you want the output of the program you started to be more than
just text. This would also need to support the message bus for each
environment, the JSON approach might be lower upfront cost.

But that is not to say that shells already supporting structured data might not
separately benefit from integration into message-buses also.

# Traditional shells
It might be possible to write helper programs for more traditional shells (e.g.
bash), especially if the `JSON_STDOUT` solution works. In this case we could
have a command, say `with-jq <command> <filter>` which could act like
`JSON_STDOUT=... <command> | jq <filter>`. Though not sure if this improves the
UX for bash at all, just putting `| jq <filter>` at the end isn't too much
trouble, though that alone would not tell the external program to output JSON.
