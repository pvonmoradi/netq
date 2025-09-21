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
| Finder                                                      | country_code | Dependencies              | Notes            |
|:------------------------------------------------------------|:------------:|:-------------------------:|:----------------:|
| [cloudflare.com](https://cloudflare.com/cdn-cgi/trace)      | ✓            | (curl\|wget)+awk          |                  |
| [ipinfo.io](https://ipinfo.io)                              | ✓            | (curl\|wget)+jq           |                  |
| [ip.network](https://ip.network)                            | ✓            | (curl\|wget)+jq           |                  |
| [ifconfig.io](https://ifconfig.io)                          | ✓            | (curl\|wget)+jq           |                  |
| [ifconfig.co](https://ifconfig.co)                          | ✓            | (curl\|wget)+jq           |                  |
| [ip-api.com](https://ip-api.com)                            | ✓            | (curl\|wget)+awk          | Plain HTTP       |
| [ipstack.com](https://ipstack.com)                          | ✓            | (curl\|wget)+jq           | Requires API_KEY |
| [bare-checkip.amazonaws.com](https://checkip.amazonaws.com) | ×            | (curl\|wget)              |                  |
| [bare-api.ipify.org](https://api.ipify.org)                 | ×            | (curl\|wget)              |                  |
| [bare-ifconfig.me](https://ifconfig.me)                     | ×            | (curl\|wget)              |                  |
| [bare-icanhazip.com](https://icanhazip.com)                 | ×            | (curl\|wget)              |                  |
| dns-google                                                  | ×            | (dig\|host\|nslookup)+awk |                  |
| dns-cloudflare                                              | ×            | (dig\|host\|nslookup)+awk |                  |
| [dns-toys](https://www.dns.toys/)                           | ×            | (dig\|host\|nslookup)+awk |                  |
| dns-akamai                                                  | ×            | (dig\|host\|nslookup)+awk |                  |


## Dependencies
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
- `netsh` : required in Windows/MSYS2 _[local, list]_


# Usage
Check `netq -h`:
```console
Usage: netq [OPTIONS] COMMAND [OPTIONS] ARGS
    netq local                  : Print local IP and network name (SSID, etc)
    netq public                 : Print public IP and country code (if possible)
    netq public -l              : List public IP finder methods
    netq public "FINDERS_CSV"   : Print public IP using passed finders
    netq public -e EXCL_REGEX   : Print public IP using filtered (BRE) finders
    netq bandwidth INTERFACE    : Watch bandwidth usage of INTERFACE
    netq check                  : Check internet connectivity
    netq check URL              : Check whether URL can be downloaded
    netq check -p               : Check internet connectivity via ping (on loop)
    netq list                   : List interfaces
    netq help                   : Show help
Options:
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
this is not accurate but I believe the network latency is the main bottleneck
here):

```shell
netq p -l | awk '{print "netq p "  $1}' | sed "s/.*/'&'/" | paste -sd' ' \
| xargs hyperfine -r3 --sort mean-time --export-markdown \
/tmp/netq-"v$(netq -V)".md
```

| Command                             |     Mean [ms] | Min [ms] | Max [ms] |     Relative |
|:------------------------------------|--------------:|---------:|---------:|-------------:|
| `netq p dns-cloudflare`             |    20.8 ± 1.3 |     19.4 |     21.6 |         1.00 |
| `netq p dns-akamai`                 |    34.9 ± 0.7 |     34.1 |     35.6 |  1.68 ± 0.11 |
| `netq p dns-google`                 |   74.3 ± 51.8 |     43.8 |    134.1 |  3.57 ± 2.50 |
| `netq p ifconfig.co`                |    84.4 ± 6.9 |     77.2 |     90.9 |  4.05 ± 0.41 |
| `netq p bare-icanhazip.com`         |    85.1 ± 1.3 |     83.8 |     86.3 |  4.09 ± 0.26 |
| `netq p cloudflare.com`             |  101.0 ± 36.2 |     76.1 |    142.5 |  4.85 ± 1.76 |
| `netq p ip-api.com`                 |  110.4 ± 54.2 |     55.3 |    163.7 |  5.30 ± 2.63 |
| `netq p bare-checkip.amazonaws.com` |   134.4 ± 2.9 |    131.2 |    137.0 |  6.46 ± 0.42 |
| `netq p bare-api.ipify.org`         |   161.2 ± 6.9 |    155.3 |    168.8 |  7.74 ± 0.58 |
| `netq p ifconfig.io`                |  167.4 ± 11.3 |    154.9 |    177.0 |  8.04 ± 0.73 |
| `netq p dns-toys`                   | 187.3 ± 110.7 |    109.7 |    314.0 |  9.00 ± 5.34 |
| `netq p ip.network`                 |  193.9 ± 53.0 |    158.1 |    254.7 |  9.31 ± 2.61 |
| `netq p bare-ifconfig.me`           | 341.7 ± 185.7 |    160.7 |    531.7 | 16.41 ± 8.98 |
| `netq p ipinfo.io`                  | 365.1 ± 128.7 |    216.5 |    440.6 | 17.54 ± 6.27 |


# Development
- linter: `shellcheck`
- formatter: `shfmt -i 4 -bn -ci -sr`

