---
layout: post
title: "The Linux Workbench: Forty Packages and the Pipe That Joins Them"
date: 2026-10-13
categories: linux tools productivity
series: "Moving to Linux"
---

I typed `cal` to check a date and the shell answered:

```
Command 'cal' not found, but can be installed with:
apt install ncal
```

It was a one-line fix, and it came in the middle of other work. That is the real cost: each missing tool is small, but each one breaks your attention, and on a new machine they come one at a time for weeks. So I did the inventory once, installed everything in one go, and wrote down what each tool is for. This post is that list, written for the version of me who will forget what `chronic` does.

## Is something wrong with the install?

No. Ubuntu labels every package with a priority. I compared the `important` and `standard` packages in the `main` archive against what was installed:

```
264 important/standard in main; 1 missing
```

The one missing package was a .NET runtime pack. `ncal` is not missing by accident; Ubuntu labels it `optional`. The base system is complete. What was missing was *my* toolset: the tools a person collects over years and stops noticing. The fix is not "install more of Ubuntu". It is to write the toolset down.

## Check the install before you run it

`apt-get install -s` simulates the install as a normal user. It resolves dependencies and reports conflicts, and changes nothing:

```
$ apt-get install -s screen htop btop tree yq bat duf ... neovim
0 upgraded, 85 newly installed, 0 to remove and 45 not upgraded.
```

No `E:` lines and no `Remv` lines means the real install will not fail on dependencies.

After the real install, a check found 13 packages still missing. They were exactly the first 13 names in the command. apt keeps a log of every command line it receives, and that log gave the answer:

```
$ grep Commandline /var/log/apt/history.log | tail -1
Commandline: apt install nethogs bmon ipmitool lftp ... neovim
```

The start of the line was lost on the copy and paste. apt installed what it received, and it reported success. Check the result, not the exit code.

## The toolbox

Grouped by the job, not by the package name.

**Lab and systems work**

| Package (command) | Use it when |
|---|---|
| [`ipmitool`](https://github.com/ipmitool/ipmitool) | You need to power-cycle a server through its BMC, read its sensors, or open a serial console over the LAN (`ipmitool sol activate`). |
| [`minicom`](https://salsa.debian.org/minicom-team/minicom), [`screen`](https://www.gnu.org/software/screen/) | You connect to a serial console: `screen /dev/ttyUSB0 115200`. `screen` is also a terminal multiplexer. |
| [`expect`](https://core.tcl-lang.org/expect/index) | A device wants an interactive login and you want a script: wait for `login:`, send the user name, and so on. |
| [`hwloc`](https://www.open-mpi.org/projects/hwloc/) (`lstopo`) | You need the CPU, cache, NUMA and PCIe layout: "which NUMA node is this NIC on?" |
| [`nvme-cli`](https://github.com/linux-nvme/nvme-cli) | You need NVMe drive health, SMART data, or the firmware version. |
| [`ltrace`](https://gitlab.com/cespedes/ltrace) | `strace` shows system calls. `ltrace` shows library calls. |
| [`iotop`](http://guichaz.free.fr/iotop/) | The disk is busy and you need to know which process. |
| [`htop`](https://htop.dev/), [`btop`](https://github.com/aristocratos/btop) | Interactive process views. `btop` adds network and disk graphs. |

**Network**

| Package (command) | Use it when |
|---|---|
| [`tshark`](https://www.wireshark.org/docs/man-pages/tshark.html) | You need Wireshark on a host with no screen. Same decoders, text output. |
| [`nmap`](https://nmap.org/) | Port scans and host discovery. |
| [`traceroute`](https://traceroute.sourceforge.net/) | You need the path. `traceroute -T -p 443` sends TCP probes that get through firewalls which drop ICMP. |
| [`arping`](https://github.com/ThomasHabets/arping) | You need to prove Layer 2 reachability, or find two hosts that use one IP address. |
| [`iftop`](https://pdw.ex-parrot.com/iftop/), [`nethogs`](https://github.com/raboof/nethogs), [`bmon`](https://github.com/tgraf/bmon) | Bandwidth per connection, per process, and per interface. |
| [`ipcalc`](https://jodies.de/ipcalc) | Subnet arithmetic. |
| [`whois`](https://github.com/rfc1036/whois) | Who owns this domain or address block. |
| [`lftp`](https://lftp.yar.ru/) | FTP/SFTP that you can script, with `mirror` for whole directories. |

**Text, pipes and automation**

| Package (command) | Use it when |
|---|---|
| [`moreutils`](https://joeyh.name/code/moreutils/) | `ts` timestamps each line, `sponge` writes a file in place at the end of a pipe, `chronic` is silent unless the command fails, `vipe` opens the middle of a pipe in your editor. |
| [`parallel`](https://www.gnu.org/software/parallel/) | GNU parallel: run one command over many inputs, across cores or hosts. |
| [`gron`](https://github.com/tomnomnom/gron) | You need to `grep` JSON. It makes one line per value, with the full path on each line. |
| [`yq`](https://github.com/kislyuk/yq) | `jq` for YAML. |
| [`pv`](https://www.ivarch.com/programs/pv.shtml) | You want a progress bar and a rate in the middle of a pipe. |
| [`entr`](https://eradman.com/entrproject/) | Run a command again each time a file changes. |
| [`inotify-tools`](https://github.com/inotify-tools/inotify-tools) | A shell script must react to file events (`inotifywait`). |
| [`direnv`](https://direnv.net/) | Each project directory needs its own environment variables, loaded on `cd`. |
| [`hyperfine`](https://github.com/sharkdp/hyperfine) | You want to compare two commands with statistics, not with one run each. |
| [`httpie`](https://httpie.io/) (`http`) | REST calls with output a person can read. |
| [`bat`](https://github.com/sharkdp/bat), [`colordiff`](https://www.colordiff.org/), [`tree`](https://oldmanprogrammer.net/source.php?dir=projects/tree), [`duf`](https://github.com/muesli/duf), [`dos2unix`](https://waterlan.home.xs4all.nl/dos2unix.html) | `cat` with colours and line numbers, coloured diff, directory tree, a clearer `df`, CRLF to LF. |
| [`tldr`](https://tldr.sh/) | You know the command but not the flags. It shows six to eight worked examples instead of the whole man page. |

## Six of them, with real output

**`ts`: a timestamp on every line of anything.** No change to the program that writes the lines:

```
$ ping -c 3 -i 0.3 127.0.0.1 | ts '%H:%M:%.S'
20:31:24.264109 PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
20:31:24.264208 64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.039 ms
20:31:24.553018 64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.049 ms
20:31:24.856059 64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.069 ms
```

**`sponge`: edit a file in place in a pipe.** The shell opens and empties the output file before `sort` reads it. `sponge` reads all of its input first, then writes:

```
$ sort -u l.txt > l.txt;            wc -l < l.txt
0
$ sort -u l.txt | sponge l.txt;     paste -sd, l.txt
a,b
```

**`chronic`: quiet when it works, loud when it fails.** This is what you want for cron jobs:

```
$ chronic true; echo rc=$?
rc=0
$ chronic sh -c 'echo boom >&2; exit 3'; echo rc=$?
rc=3
boom
```

**`gron`: `grep` for JSON.** Many Linux tools can write JSON (`ip -j`, `lsblk -J`, `ethtool --json`). `gron` makes that JSON searchable with the tool you already know:

```
$ ip -j addr show lo | gron | grep -E 'ifname|local|prefixlen'
json[0].addr_info[0].local = "127.0.0.1";
json[0].addr_info[0].prefixlen = 8;
json[0].addr_info[1].local = "::1";
json[0].addr_info[1].prefixlen = 128;
json[0].ifname = "lo";
```

Each line is the path to the value, so the `grep` result also tells you how to write the `jq` query.

**`ipcalc`: the subnet, in binary.**

```
$ ipcalc 10.1.0.0/22
Network:   10.1.0.0/22          00001010.00000001.000000 00.00000000
HostMin:   10.1.0.1             00001010.00000001.000000 00.00000001
HostMax:   10.1.3.254           00001010.00000001.000000 11.11111110
Broadcast: 10.1.3.255           00001010.00000001.000000 11.11111111
Hosts/Net: 1022                  Class A, Private Internet
```

**`hyperfine`: measure, do not believe.** I expected ripgrep to win against grep. On a small file, it did not:

```
$ hyperfine -N --warmup 3 --runs 20 'grep -c tcp /etc/services' 'rg -c tcp /etc/services'
Benchmark 1: grep -c tcp /etc/services
  Time (mean ± σ):     813.8 µs ± 152.8 µs
Benchmark 2: rg -c tcp /etc/services
  Time (mean ± σ):       1.7 ms ±   0.3 ms
Summary
  grep -c tcp /etc/services ran
    2.15 ± 0.52 times faster than rg -c tcp /etc/services
```

`/etc/services` is about 12 KB. At that size, the time to start the program is most of the total, and ripgrep's start-up work (thread pool, ignore-file rules) costs more than it saves. ripgrep wins on large trees, which is the job it was built for. One measurement shows you the limit of a rule of thumb.

## Traps I found

| Trap | What happens | Fix |
|---|---|---|
| `bat`, `fd-find` | Debian already had other programs called `bat` and `fd`, so the commands are `batcat` and `fdfind`. | Alias them in `~/.bash_aliases`. |
| `yq` | The apt package is the Python `yq` (a `jq` wrapper). The popular Go `yq` is a different program with a different syntax. | Choose one. For the Go version, use `snap install yq` or the release binary. |
| `tldr` | On Ubuntu 24.04, both clients in apt fail on first update: `tldr` (Haskell) and `tealdeer` 1.6.1 both download `tldr.sh/assets/tldr.zip`, which is now an HTML page that redirects to GitHub. The error is `invalid Zip archive`. | `uv tool install tldr` (the Python client, 3.4.3), then `tldr --update` → `Updated cache for language en: 7580 entries`. |
| `parallel` | `moreutils` also has a `parallel`. | Nothing to do: the GNU package moves the other one to `parallel.moreutils`. Check with `parallel --version`. |
| `tshark` | The install asks whether users other than root can capture packets. | If you say yes, also run `sudo usermod -aG wireshark $USER` and log in again. |
| `direnv` | Installed but does nothing. | Add `eval "$(direnv hook bash)"` to `~/.bashrc`. |
| `git-lfs` | Installed but not active. | `git lfs install`, one time. |

## What I carried over from Windows

The list above came from "what do I miss today". A second source is better: what I had already chosen, over years, on Windows. Both package managers can export what they installed, so the old machine had left me two inventories:

```
winget export -o winget-packages.json    # 141 package IDs
scoop export > scoop-packages.json       # 59 apps
```

Many winget entries were not command-line tools: runtimes, GUI applications, and programs that exist only on Windows. For each one that *was* a tool, I wrote down the command it gives you (`difftastic` → `difft`, `miller` → `mlr`), then checked for that command on the Linux machine: on `PATH`, in `~/.local/bin`, `~/.cargo/bin` and `~/go/bin`, and in `snap list` and `flatpak list`. A package name is not proof; a command you can run is.

The result had four groups:

| Result | Tools | What I did |
|---|---|---|
| Already on Linux | [`chezmoi`](https://www.chezmoi.io/), [`ghq`](https://github.com/x-motemen/ghq), [`gh`](https://cli.github.com/), [`go`](https://go.dev/), [`node`](https://nodejs.org/), [`bun`](https://bun.sh/), [`uv`](https://docs.astral.sh/uv/), [`typst`](https://typst.app/), [`sqlite3`](https://sqlite.org/cli.html), [`aria2c`](https://aria2.github.io/), [`7z`](https://www.7-zip.org/), [`graphviz`](https://graphviz.org/), [`imagemagick`](https://imagemagick.org/), [`gitleaks`](https://github.com/gitleaks/gitleaks), [`mise`](https://mise.jdx.dev/), [`wezterm`](https://wezterm.org/), [`plocate`](https://plocate.sesse.net/) (in place of Everything), [`qdirstat`](https://github.com/shundhammer/qdirstat) (in place of WinDirStat), and more | Nothing. The earlier setup had already covered them. |
| Missing, in apt | [`lnav`](https://lnav.org/), [`miller`](https://miller.readthedocs.io/), [`zoxide`](https://github.com/ajeetdsouza/zoxide), [`eza`](https://github.com/eza-community/eza), [`ffmpeg`](https://ffmpeg.org/), [`pandoc`](https://pandoc.org/), [`qpdf`](https://qpdf.sourceforge.io/), [`pdftk-java`](https://gitlab.com/pdftk-java/pdftk) | Installed with apt, after an `apt-get install -s` dry run with no errors. |
| Missing, not in apt | [`difftastic`](https://github.com/Wilfred/difftastic), [`dust`](https://github.com/bootandy/dust), [`dua`](https://github.com/Byron/dua-cli), [`zellij`](https://zellij.dev/), [`yazi`](https://yazi-rs.github.io/), [`starship`](https://starship.rs/), [`mcfly`](https://github.com/cantino/mcfly), [`aichat`](https://github.com/sigoden/aichat) | Installed with [mise](https://mise.jdx.dev/), which downloads the upstream release binaries. |
| Not needed | kitty, alacritty, Double Commander, FileZilla, PowerShell, MiKTeX, and similar | Another tool already does the job: WezTerm, `lf`, `lftp`, bash, Typst. |

The mise group takes one command. `-g` writes the tools to `~/.config/mise/config.toml`, so the list is a file, and that file goes into the dotfiles repo:

```bash
mise use -g aichat difftastic dua dust starship yazi zellij aqua:cantino/mcfly
```

`mcfly` is not in mise's short-name registry, so the command names the backend and the repository directly. Before the install, `mise latest <tool>` checks that each name resolves to a release (`difftastic 0.71.0`, `zellij 0.45.1`, and so on).

Five of the new tools:

| Tool | Use it when |
|---|---|
| [`lnav`](https://lnav.org/) | You read logs. It merges several log files by timestamp, and you can query them with SQL: `;SELECT log_level, count(*) FROM syslog_log GROUP BY 1`. |
| [`miller`](https://miller.readthedocs.io/) (`mlr`) | You have CSV or JSON and want `awk` by field name: `mlr --icsv --opprint sort -nr count then head -n 5 data.csv`. |
| [`difftastic`](https://github.com/Wilfred/difftastic) (`difft`) | You want a diff by syntax tree, not by line. A changed indent or a wrapped line is not reported as a change. Set it as `git diff`'s external diff. |
| [`zoxide`](https://github.com/ajeetdsouza/zoxide) (`z`) | `cd` that remembers. `z repo` goes to the directory you visit most whose path matches "repo". |
| [`dust`](https://github.com/bootandy/dust), [`dua`](https://github.com/Byron/dua-cli) | `du` as a tree with bars (`dust`), or as an interactive cleaner (`dua i`). |

Sixteen tools that I had chosen on Windows were missing on Linux. None of them gave an error, because a tool that is absent cannot fail; I would have found each one only on the day I typed its name. The inventory found them all in one pass.

## What actually changes compared with Windows

Most of these tools run on Windows too. nmap, Wireshark, neovim, hyperfine and bat all have Windows builds, and I spent a whole post [getting Unix tools to work in PowerShell]({% post_url 2026-03-03-getting-unix-tools-to-work-in-powershell %}). The difference is not which tools exist.

The difference is what joins them. On Linux, every tool on this list reads text and writes text, so any one can feed any other one. `tshark` into `ts` into `grep` into `parallel` is a new tool, made in ten seconds, that nobody had to plan for. PowerShell joins its own commands with objects, which is powerful inside PowerShell, but most native Windows tools are separate programs, and many are GUIs. On Linux, text is the one interface that every tool uses.

This idea is old. In 1978 Doug McIlroy wrote, in the *Bell System Technical Journal*: "Expect the output of every program to become the input to another, as yet unknown, program." Peter Salus later summarised it (1994) as "Write programs to handle text streams, because that is a universal interface." What McIlroy did not predict is that the "as yet unknown program" would one day be an LLM agent. An agent works by running commands and reading text. A workbench of small text tools is also the best workbench for an agent, for the same reason it was good for people in 1978.

## Keep the list as code

A list in a blog post goes out of date. A list in a dotfiles repo is the setup. I keep dotfiles in [chezmoi](https://www.chezmoi.io/user-guide/use-scripts-to-perform-actions/), which runs a `run_onchange_` script again each time its contents change. So the workflow to add a tool is: add one name to the list, then `chezmoi apply`. Tools that apt does not have go into mise's `config.toml`, which chezmoi also tracks.

```bash
#!/usr/bin/env bash
# run_onchange_before_10-install-apt-packages.sh
set -euo pipefail
[ "$(uname -s)" = Linux ] || exit 0

PKGS=( ncal tree htop btop screen ncdu duf pv moreutils parallel entr direnv
       bat ripgrep fd-find fzf jq yq gron colordiff dos2unix inotify-tools neovim
       universal-ctags hyperfine httpie traceroute mtr-tiny nmap tshark arping
       whois iftop nethogs bmon ipcalc ipmitool lftp minicom expect socat
       ltrace iotop nvme-cli hwloc git-lfs )

command -v uv >/dev/null && uv tool install --quiet tldr   # apt clients are broken

missing=()
for p in "${PKGS[@]}"; do
  dpkg-query -W -f='${db:Status-Abbrev}' "$p" 2>/dev/null | grep -q '^ii' || missing+=("$p")
done
[ ${#missing[@]} -eq 0 ] && exit 0
sudo apt-get update -qq
sudo apt-get install -y "${missing[@]}"
```

It asks for `sudo` only when something is missing. On Windows, the same repo holds a `winget.json` for the same purpose. The next time a command is missing, it costs one line in a file, and the fix also applies to the next machine.

## Source

- [The Unix philosophy](https://en.wikipedia.org/wiki/Unix_philosophy): McIlroy 1978, Salus 1994
- [moreutils](https://joeyh.name/code/moreutils/): `ts`, `sponge`, `chronic`, `vipe`
- [GNU parallel](https://www.gnu.org/software/parallel/)
- [gron](https://github.com/tomnomnom/gron)
- [entr](https://eradman.com/entrproject/)
- [hyperfine](https://github.com/sharkdp/hyperfine)
- [direnv](https://direnv.net/)
- [yq (Python)](https://github.com/kislyuk/yq) and [yq (Go)](https://github.com/mikefarah/yq)
- [tldr-pages](https://github.com/tldr-pages/tldr) and its [Python client](https://github.com/tldr-pages/tldr-python-client)
- [chezmoi scripts](https://www.chezmoi.io/user-guide/use-scripts-to-perform-actions/)
- [mise](https://mise.jdx.dev/), [lnav](https://lnav.org/), [Miller](https://miller.readthedocs.io/), [difftastic](https://github.com/Wilfred/difftastic)
