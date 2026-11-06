# vhost-recurse

**Recursive virtual host brute forcing with [ffuf](https://github.com/ffuf/ffuf).**

`vhost-recurse` finds the vhosts on a target, then automatically fuzzes each new vhost one level deeper (`FUZZ.<found-vhost>`), and keeps going until a level finds nothing new. It turns a manual, repetitive loop into one command.

```
example.com
├── admin.example.com [200] 22 B
└── dev.example.com [200] 29 B
    └── api.dev.example.com [200] 26 B
        └── test.api.dev.example.com [200] 13 B
```

## Why

The usual vhost workflow looks like this:

1. Brute force vhosts on the target
2. Add what you found to `/etc/hosts`
3. Brute force again under each new vhost (`FUZZ.dev.example.com`)
4. Repeat until nothing new shows up

Steps 3 and 4 are easy to skip or lose track of. Nested vhosts such as `api.dev.example.com` are exactly the ones a single scan misses. This tool runs the whole loop for you.

## Features

- **Recursive by design**: every vhost found becomes a new base to fuzz, up to a depth you choose
- **Reliable filtering**: measures the default response itself instead of relying on ffuf's `-ac`, which can mislabel real hits when running with many threads
- **Fails loudly**: checks the target is reachable before fuzzing, so a wrong IP or port doesn't look like "no vhosts found"
- **Separate recursion wordlist** (`-W`): use a big list for the first level and a small one for the deeper levels to keep runs fast
- **gobuster-style target** (`-u http://example.com:PORT`): takes the domain and port straight from the URL
- **Live progress**: progress bar with request rate, elapsed time, and error count, plus a little ASCII runner
- **Ready-to-use output**: a vhost list, a tree view, and a line you can append straight to `/etc/hosts` (or let the tool add it with `-a`)

## Requirements

- `bash` 4+
- [`ffuf`](https://github.com/ffuf/ffuf)
- `jq`
- `curl`
- A wordlist, e.g. [SecLists](https://github.com/danielmiessler/SecLists) (`Discovery/DNS/`)

On Kali / Parrot:

```sh
sudo apt install ffuf jq curl seclists
```

## Installation

```sh
git clone https://github.com/raphwaf/vhost-recurse.git
cd vhost-recurse
chmod +x vhost-recurse.sh
```

Optionally, put it on your `PATH`:

```sh
sudo ln -s "$PWD/vhost-recurse.sh" /usr/local/bin/vhost-recurse
```

## Usage

```
vhost-recurse.sh -u <url> [options] [-- <extra ffuf args>]
vhost-recurse.sh -i <ip> -d <domain> [-p <port>] [options] [-- <extra ffuf args>]
```

### Target

| Option | Description |
|---|---|
| `-u <url>` | Target URL, like gobuster: `http://example.com:8080`. The hostname becomes the base domain and is resolved to an IP via `/etc/hosts` or DNS. |
| `-i <ip>` | Target IP address |
| `-d <domain>` | Base domain (e.g. `example.com`). Overrides the hostname from `-u`. |
| `-p <port>` | Port (default: from `-u`, otherwise 80 / 443) |
| `-s <scheme>` | `http` or `https` (default: from `-u`, otherwise `http`) |

### Options

| Option | Description | Default |
|---|---|---|
| `-w <wordlist>` | Wordlist for the first level | `subdomains-top1million-5000.txt` |
| `-W <wordlist>` | Wordlist for depth 2 and deeper | same as `-w` |
| `-D <depth>` | Max recursion depth (`1` = no recursion) | `2` |
| `-t <threads>` | ffuf threads | `40` |
| `-o <dir>` | Output directory | `./vhosts_<domain>_<timestamp>` |
| `-a` | Add found vhosts to `/etc/hosts` (uses `sudo` if not root) | off |
| `-h` | Show help | |

Anything after `--` is passed straight to ffuf. If you pass your own filter there (`-fs`, `-fw`, `-fc`, `-ac`, ...), it replaces the tool's automatic baseline.

### Environment variables

| Variable | Description |
|---|---|
| `FFUF_BIN` | Path to ffuf, if it isn't on your `PATH` |
| `NO_COLOR` | Set to any value to disable colors |

## Examples

Same target and wordlist as a typical gobuster run, with a smaller list for the recursion:

```sh
./vhost-recurse.sh -u http://example.com:8080 -t 100 \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
    -W /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

By IP and domain, three levels deep, adding results to `/etc/hosts`:

```sh
./vhost-recurse.sh -i 192.168.1.50 -d example.com -p 8080 -D 3 -a
```

No recursion (one level, like gobuster):

```sh
./vhost-recurse.sh -u http://example.com:8080 -D 1
```

Your own ffuf filter instead of the automatic baseline:

```sh
./vhost-recurse.sh -u http://example.com:8080 -- -fs 986
```

## Output

Everything is saved in the output directory:

| File | Contents |
|---|---|
| `vhosts.txt` | Every vhost found, one per line |
| `hosts_entries.txt` | One `/etc/hosts` line: `<ip> <domain> <vhost> <vhost> ...` |
| `raw/<base>.json` | ffuf's JSON output for each scan |
| `raw/<base>.log` | The exact ffuf command that ran, plus its output |

To add the results to `/etc/hosts` yourself:

```sh
sudo tee -a /etc/hosts < vhosts_example.com_*/hosts_entries.txt
```

## How it works

Vhost brute forcing doesn't use DNS. Every request goes to the **same IP and port**, and only the HTTP `Host` header changes:

```
GET / HTTP/1.1
Host: dev.example.com
```

The web server picks which site to serve based on that header. A name that returns something different from the default response is a vhost. That's why the tool can fuzz under a vhost it found a second ago, without it being in `/etc/hosts`.

For each base, the tool:

1. Requests a few random, non-existent names (`<random>.<base>`) to learn the **default response size**
2. Runs ffuf with `Host: FUZZ.<base>`, filtering out that size
3. Adds every new hit to the queue for the next depth level

It stops when a level finds nothing new, or when it reaches `-D`.

## Tips

- **Big first, small after.** Each vhost found costs another full scan. A large `-w` with a small `-W` gives you good coverage without multiplying the run time.
- **Errors in the progress bar mean missed vhosts.** If the error count climbs, lower the threads with `-t`.
- **Dynamic default pages.** If the default response changes size between requests, the tool falls back to ffuf's `-ac` and warns you. Check those results manually, or pass your own filter after `--`.
- **HTTPS and SNI.** With HTTPS, some servers choose the site from the TLS SNI field rather than the `Host` header. Connecting by IP may then return the wrong site, and results can be incomplete.

## Disclaimer

This tool is intended for authorized security testing, CTFs, and lab environments only. Only run it against systems you own or have explicit permission to test.