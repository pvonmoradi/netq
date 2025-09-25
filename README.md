# netq
A handy POSIX-compliant utility script to quickly fetch common network
parameters. The script is written with portablity among GNU/Linux distros in
mind. Hence, subcommands have numerous fallback tools and remote APIs to rely on
for their function.

`netq` is not designed to be an alternative to `ip` and the like. It should be
thought of as a _helper script_ to be used in other scripts and tools in
accordance with the Unix philosophy.

The output is produced on a best-effort basis, and may differ a bit dependeny on
the availabilty of dependencies on host OS or the capabilities of remote APIs.

## Features
- Query local (private) IP and an indentifier name for the corresponding network
  (SSID, iface_name, etc)
- Query public IP, with support of many HTTP-based or DNS-based ip-finder
  methods
- Watch bandwidth usage of an interface
- Supports IPv4 and IPv6

## Supported IP-finder services
| Finder                                                      | Country | IPv6 | Dependencies              | Notes            |
|:------------------------------------------------------------|:-------:|:----:|:-------------------------:|:----------------:|
| [cloudflare.com](https://cloudflare.com/cdn-cgi/trace)      | ✓       | ✓    | (curl\|wget)+awk          |                  |
| [ipinfo.io](https://ipinfo.io)                              | ✓       | ✓    | (curl\|wget)+jq           |                  |
| [ip.network](https://ip.network)                            | ✓       | ✓    | (curl\|wget)+jq           |                  |
| [ifconfig.io](https://ifconfig.io)                          | ✓       | ✓    | (curl\|wget)+jq           |                  |
| [ifconfig.co](https://ifconfig.co)                          | ✓       | ✓    | (curl\|wget)+jq           |                  |
| [ip-api.com](https://ip-api.com)                            | ✓       | ×    | (curl\|wget)+awk          | Plain HTTP       |
| [ipstack.com](https://ipstack.com)                          | ✓       | ×    | (curl\|wget)+jq           | Requires API_KEY |
| [bare-checkip.amazonaws.com](https://checkip.amazonaws.com) | ×       | ×    | (curl\|wget)              |                  |
| [bare-ipify.org](https://ipify.org)                         | ×       | ✓    | (curl\|wget)              |                  |
| [bare-ifconfig.me](https://ifconfig.me)                     | ×       | ✓    | (curl\|wget)              |                  |
| [bare-icanhazip.com](https://icanhazip.com)                 | ×       | ✓    | (curl\|wget)              |                  |
| dns-google                                                  | ×       | ✓    | (dig\|host\|nslookup)+awk |                  |
| dns-cloudflare                                              | ×       | ✓    | (dig\|host\|nslookup)+awk |                  |
| [dns-toys](https://www.dns.toys/)                           | ×       | ✓    | (dig\|host\|nslookup)+awk |                  |
| dns-akamai                                                  | ×       | ×    | (dig\|host\|nslookup)+awk |                  |

## Installation
```shell
wget https://github.com/pvonmoradi/netq/raw/refs/heads/master/netq -O ~/bin/netq
# or
wget https://github.com/pvonmoradi/netq/raw/refs/heads/master/netq -O /usr/local/bin/netq
```

## Dependencies
The script is expected to just work on typical GNU/Linux systems, without
needing to install non-default tools.

The following lists `netq` dependencies and the _[command]_ they are used in. Not
all of these tools are required. The | between entries means existence of one
tool is enough. Not all public IP finders need every _[public]_ tool. Refer to the
table above.

- `ip | ifconfig | nmcli | hostname` : for getting local ip _[local, list]_
- `curl | wget` : for getting content from remote API _[public, check]_
- `jq` : for processing json _[public]_
- `dig | host | nslookup` : for DNS requests _[public]_
- `ping` : for pinging remote IP _[check]_
- `awk bc date grep tr uname`
- `netsh netstat` : only required in Windows/MSYS2 _[local, list, bandwidth]_


# Usage
Check `netq -h`:
```console
Usage: netq [OPTIONS] COMMAND [OPTIONS] ARGS
    netq local                  : Print local IP and network name (SSID, etc)
    netq public                 : Print public IP and country code (if possible)
    netq public -l              : List public IP finder methods
    netq public "FINDERS_CSV"   : Print public IP using passed finders
    netq public -e EXCL_REGEX   : Print public IP using filtered (BRE) finders
    netq public -r              : Print raw response from public IP finder
    netq bandwidth INTERFACE    : Watch bandwidth usage of INTERFACE
    netq check                  : Check internet connectivity
    netq check URL              : Check whether URL can be downloaded
    netq check -p               : Check internet connectivity via ping (on loop)
    netq list                   : List interfaces
    netq help                   : Show help
Options:
    -4: IPv4 mode (default)
    -6: IPv6 mode
    -q: Suppress all logs even errors
    -v: Enable debug logs
    -V: Print version
    -h: Show help
Command aliases:
    local: l, public: p, bandwidth: b|bw, check: c, list: ls
```

## Examples
- `netq local` : Print local IP and connection name. Suitable for figuring out
  what the local IP of this machine is, so other devices on local network could
  access http servers running on this machine
  
- `netq public` : Print public IP address of the machine, as seen by remote
  machines on the internet. Also prints the two-character country code if the
  _finder_ method supports it
  
- `netq p -l` : (used alias p for public) List public IP finder methods

- `netq p -e 'dns-*'` : Exclude all dns-based IP finder methods and print the
  public IP. Note that regualr expression dialect should be in [POSIX Basic
  Regular Expression
  (BRE)](https://en.wikibooks.org/wiki/Regular_Expressions/POSIX_Basic_Regular_Expressions)
  
- `netq p -e 'dns-*\|bare-*' -l` : Exclude all methods that don't support
  extraction of country, and list the resulting finder methods (note the escape
  char before |). The `-l` here is useful for forming up `-e`
  
- `netq -v p "dns-google,cloudflare.com"` : Only use the passed finder methods
  in the specified priority (note CSV format). The `-e` switch in this case
  would act on the passed finders. The global `-v` switch enables debug logs
  
- `netq p -r "ip-api.com"` : Print the raw response of the remote API without
  any postprocessing. Useful for manually extracting parameters other than IP or
  country. Output varies depending on finder services

- `netq -6 p` : Print public IPv6 of the machine
  
- `netq ls` : List available interfaces

- `netq bw wlp3s0` : Watch bandwidth usage for the passed wireless
  device. Output is updated on a single line, and Rx/Tx units would scale
  automatically

- `netq check` : Check internet connectivity by downloading a default website
  (https://www.google.com). This is useful to detect whether the machine is
  behind captive portals or not. Could be used as a pre-condition in systemd
  services that depend on internet connectivity

- `netq c "https://example.com"` : Same as above, but download the passed
  URL instead
  
- `netq c -p` : Continuously ping a default IP (8.8.8.8) and halt if ping is
  successful


# Notes
For public IP finding, naturally, DNS-based methods (UDP) are faster than
HTTP-based ones. What follows is a rough benchmark of supported public IP
finders (note that different finders use different post-processing tools hence
`-r` is used in this benchmark. Anyway, I believe network latency is the main
bottleneck here):

```shell
netq p -l | awk '{print "netq p -r "  $1}' | sed "s/.*/'&'/" | paste -sd' ' \
| xargs hyperfine -r3 -N --sort mean-time --export-markdown \
/tmp/netq-"v$(netq -V)".md
```

| Command                                |    Mean [ms] | Min [ms] | Max [ms] |     Relative |
|:---------------------------------------|-------------:|---------:|---------:|-------------:|
| `netq p -r dns-cloudflare`             |   38.6 ± 4.9 |     33.5 |     43.2 |         1.00 |
| `netq p -r dns-google`                 |   45.2 ± 6.5 |     41.3 |     52.6 |  1.17 ± 0.22 |
| `netq p -r ip-api.com`                 |   54.1 ± 3.4 |     50.5 |     57.2 |  1.40 ± 0.20 |
| `netq p -r cloudflare.com`             |   62.3 ± 9.7 |     51.2 |     69.5 |  1.61 ± 0.32 |
| `netq p -r bare-icanhazip.com`         |  84.5 ± 15.1 |     68.8 |     98.9 |  2.19 ± 0.48 |
| `netq p -r ifconfig.co`                |   89.2 ± 9.7 |     78.1 |     95.1 |  2.31 ± 0.38 |
| `netq p -r bare-checkip.amazonaws.com` |  96.6 ± 20.6 |     81.8 |    120.1 |  2.50 ± 0.62 |
| `netq p -r dns-akamai`                 |  107.7 ± 3.0 |    104.3 |    109.7 |  2.79 ± 0.36 |
| `netq p -r dns-toys`                   | 132.8 ± 20.0 |    109.9 |    146.2 |  3.44 ± 0.68 |
| `netq p -r ifconfig.io`                |  160.7 ± 6.7 |    154.7 |    167.9 |  4.16 ± 0.55 |
| `netq p -r bare-ifconfig.me`           | 165.9 ± 34.7 |    145.9 |    206.0 |  4.30 ± 1.05 |
| `netq p -r ip.network`                 | 166.6 ± 21.6 |    146.3 |    189.4 |  4.32 ± 0.78 |
| `netq p -r ipinfo.io`                  | 167.6 ± 16.3 |    148.8 |    178.1 |  4.34 ± 0.69 |
| `netq p -r bare-ipify.org`             | 433.6 ± 24.1 |    407.7 |    455.2 | 11.23 ± 1.55 |

# Integrations
- [`bandwidth-monitor`](contrib/bandwidth-monitor/bandwidth-monitor) : useful
  for checking desktop traffic usage, e.g. ensuring whether uploading of a file
  to a web service is progressing

# Development
- linter: `shellcheck`
- formatter: `shfmt -i 4 -bn -ci -sr`

