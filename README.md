# dig

A local `dig` wrapper for macOS and Linux. The actual query is still sent by the system `dig` binary at `/usr/bin/dig`. This project only appends a local TXT record after your own terminal output, to make local testing easier.

It does not change DNS, does not write zone files, and does not touch any cloud provider.

## Behavior

**Pin a TXT record to a domain, and every subsequent query returns it**

```sh
dig +txt=<value> TXT <name>
dig +txt=<value> +ttl=600 TXT <name>
```

- `name`: domain name, e.g. `app.local`
- `value`: the content of this TXT record, written after `+txt=`. Quote it if it contains spaces
- `+ttl=<seconds>`: display-only TTL, default 600

```sh
dig +txt=env=staging TXT app.local
dig +short TXT app.local              # "env=staging"
dig +txt='hello world' TXT app.local  # value contains spaces
dig +txt= TXT app.local               # remove the record for this domain
```

It can also be written as positional arguments:

```sh
dig app.local env=staging
dig app.local -- "hello world"
```

It only applies to query types `TXT` / `ANY`; querying `A` will not show it. Setting `example.com` also matches queries for `x.example.com`. `+txt=` does not appear in `dig -h`; this argument is stripped before being passed to system `dig`.

A non-hexadecimal `+cookie=<value>` likewise sets `value`. A real hexadecimal cookie is passed through to system `dig` unchanged.

In full output, system `dig`'s HEADER, `ANSWER:` count, and `MSG SIZE rcvd` **remain exactly as the network response**. The local TXT is written after that output, separated by a blank line, for example:

```text
;; MSG SIZE  rcvd: 100

app.local.	600	IN	TXT	"env=staging"
```

With `+short`, only one additional quoted line appears. `ANSWER SECTION` is not rewritten, and no provenance note is printed to stderr.

A second query for the same domain **does not** append a `_zcode-verify` marker again. The local machine still records the query count, but the output is unchanged.

![DNS verification passed, TXT results from three resolvers match](docs/digdns.png)

## Installation

Supported on macOS and Linux. Requires `/usr/bin/dig` and `/usr/bin/python3`, with no pip dependencies. On Linux, install dig and Python 3 first:

```sh
# Debian / Ubuntu
sudo apt install dnsutils python3

# Fedora / RHEL
sudo dnf install bind-utils python3

# Arch Linux
sudo pacman -S bind python
```

```sh
git clone https://github.com/YusukeJustinNakajima/dig.git
cd dig
./scripts/install.sh
export PATH="$HOME/.local/bin:$PATH"
command -v dig    # should be ~/.local/bin/dig
```

The install script backs up any existing `~/.local/bin/dig` first. If `command -v dig` is not this path, the wrapper will not take effect at all. On Linux the installer records the system dig checksum with `sha256sum` (or `shasum`); macOS-specific code-signature metadata is skipped.

An already-open zsh may still remember the old path; run `rehash` once.

## Restore

```sh
./scripts/restore.sh
```

This restores the file that existed before installation, which is not necessarily the same as the system `/usr/bin/dig`. To bypass the wrapper temporarily, call `/usr/bin/dig` directly.

Do not manually unpack the backup directory.

## State

Counters and pinned TXT records are stored in:

```text
~/.cache/dig-zcode-wrapper
```

This is local test state, not DNS evidence. Do not commit it, and do not use it to verify domain ownership.

## Verification

```sh
make verify
```

## Limitations

- Works on macOS and Linux, and only when `~/.local/bin` precedes other entries in PATH
- The extra TXT exists only in the output of this command
- It does not replace a recursive resolver or authoritative server, and does not change the DNS network path used by `/usr/bin/dig`
