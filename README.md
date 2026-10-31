# Tor + Automatic Circuit Rotation on Arch Linux and Kali Linux

This guide explains how to install and configure Tor on **Arch Linux** or Kali Linux, then set up a single `tor-rotate` script that:

- runs a command through Tor using `torsocks`;
- requests a new Tor circuit at startup;
- requests another circuit rotation every 10 seconds while the command is running;
- returns the exit code of the executed command.

## Installation on Arch Linux

```bash
sudo pacman -Syu
sudo pacman -S tor torsocks openbsd-netcat
```

Enable and start the service:

```bash
sudo systemctl enable --now tor
sudo systemctl status tor
```

## Installation on Kali Linux

```bash
sudo apt update
sudo apt install tor torsocks netcat-openbsd
```

Enable and start the service:

```bash
sudo systemctl enable --now tor
sudo systemctl status tor
```

## Configuring `/etc/tor/torrc`

The configuration is the same on Arch Linux and Kali Linux.

```bash
sudo nano /etc/tor/torrc
```

Add or check the following settings:

```conf
SocksPort 127.0.0.1:9050
SafeSocks 1
TestSocks 1

ControlPort 127.0.0.1:9051
CookieAuthentication 1
DataDirectoryGroupReadable 1
CookieAuthFileGroupReadable 1
```

Restart Tor:

```bash
sudo systemctl restart tor
sudo systemctl status tor
```

You should see the following messages in the logs:

```text
Opening Socks listener on 127.0.0.1:9050
Opening Control listener on 127.0.0.1:9051
```

## Accessing the Tor Authentication Cookie

### Arch Linux

Add your user to the `tor` group:

```bash
sudo usermod -aG tor "$USER"
```

### Kali Linux

Add your user to the `debian-tor` group:

```bash
sudo usermod -aG debian-tor "$USER"
```

On either distribution, reboot your computer, then check your groups:

```bash
id
```

You should see the corresponding Tor group:

```text
Arch Linux: groups=...,tor
Kali Linux: groups=...,debian-tor
```

The cookie is located here:

```text
/var/lib/tor/control_auth_cookie
```

Check that you can read it:

```bash
cat /var/lib/tor/control_auth_cookie >/dev/null && echo "Access OK"
```

If access is denied, apply the following commands for your distribution.

### Arch Linux

```bash
sudo chgrp tor /var/lib/tor
sudo chmod 750 /var/lib/tor
sudo chgrp tor /var/lib/tor/control_auth_cookie
sudo chmod 640 /var/lib/tor/control_auth_cookie
```

### Kali Linux

```bash
sudo chgrp debian-tor /var/lib/tor
sudo chmod 750 /var/lib/tor
sudo chgrp debian-tor /var/lib/tor/control_auth_cookie
sudo chmod 640 /var/lib/tor/control_auth_cookie
```

## Setting Up the `tor-rotate` Script

The script is the same on Arch Linux and Kali Linux.

You can copy the existing script from this repository into `~/.local/bin/`, or create it manually as shown below.

```bash
mkdir -p ~/.local/bin
nano ~/.local/bin/tor-rotate
```

Paste the following content:

```bash
#!/usr/bin/env bash
set -euo pipefail

COOKIE=/var/lib/tor/control_auth_cookie
HOST=127.0.0.1
PORT=9051
DELAY="${TOR_ROTATE_DELAY:-10}"

if [[ $# -eq 0 ]]; then
  echo "Usage: tor-rotate COMMAND [arguments...]" >&2
  exit 1
fi

if (( DELAY < 10 )); then
  DELAY=10
fi

if [[ ! -r "$COOKIE" ]]; then
  echo "Error: Tor authentication cookie is not readable: $COOKIE" >&2
  exit 1
fi

rotate() {
  local cookie_hex response
  cookie_hex=$(xxd -p "$COOKIE" | tr -d '\n')

  response=$(printf 'AUTHENTICATE %s\r\nSIGNAL NEWNYM\r\nQUIT\r\n' "$cookie_hex" \
    | timeout 5 nc "$HOST" "$PORT")

  if grep -q '^250 OK' <<<"$response"; then
    echo "[tor-rotate] $(date '+%H:%M:%S') — New circuit requested."
  else
    echo "[tor-rotate] Tor error:" >&2
    echo "$response" >&2
    return 1
  fi
}

rotate

echo "[tor-rotate] Running through Tor: $*"

torsocks "$@" &
CMD_PID=$!

while kill -0 "$CMD_PID" 2>/dev/null; do
  sleep "$DELAY"
  rotate || true
done

set +e
wait "$CMD_PID"
STATUS=$?
set -e

exit "$STATUS"
```

Make the script executable:

```bash
chmod +x ~/.local/bin/tor-rotate
```

Make sure `~/.local/bin` is included in your `PATH`.

For Zsh:

```bash
echo "$PATH" | grep -q "$HOME/.local/bin" || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

If you use Bash instead:

```bash
echo "$PATH" | grep -q "$HOME/.local/bin" || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## Usage

```bash
tor-rotate COMMAND [arguments...]
```

The command is automatically launched through `torsocks`, which redirects supported network connections through Tor.

## Example 1: Check That Tor Is Working

```bash
tor-rotate curl [https://check.torproject.org/api/ip](https://check.torproject.org/api/ip)
```

Expected output:

```json
{"IsTor":true,"IP":"x.x.x.x"}
```

## Example 2: Test Automatic Circuit Rotation

This command sends six requests with a five-second pause between them. The total runtime also includes the time spent waiting for each response.

```bash
tor-rotate bash -c 'for i in {1..6}; do curl -s [https://check.torproject.org/api/ip](https://check.torproject.org/api/ip); echo; sleep 5; done'
```

You should see several messages similar to:

```text
[tor-rotate] 19:30:07 — New circuit requested.
[tor-rotate] 19:30:17 — New circuit requested.
[tor-rotate] 19:30:27 — New circuit requested.
```

The IP address may change, but a different address is not guaranteed after every rotation. `NEWNYM` requests new circuits; it does not guarantee a different exit relay.

## Example 3: Run ffuf Through Tor

```bash
tor-rotate ffuf -u http://target/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

Tor can significantly slow down fuzzing. Reduce the number of threads to limit load:

```bash
tor-rotate ffuf -u http://target/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10
```

Use this only against systems you own or have explicit permission to test.

## Changing the Rotation Interval

By default, the script requests a rotation every 10 seconds.

To use a 15-second interval:

```bash
TOR_ROTATE_DELAY=15 tor-rotate curl [https://check.torproject.org/api/ip](https://check.torproject.org/api/ip)
```

The script enforces a minimum interval of 10 seconds because Tor rate-limits closely spaced `NEWNYM` requests.

## Troubleshooting

### `nc: command not found`

```bash
# Arch Linux
sudo pacman -S openbsd-netcat

# Kali Linux
sudo apt install netcat-openbsd
```

### Tor Authentication Cookie Is Not Readable

Check:

```bash
id
ls -ld /var/lib/tor
ls -l /var/lib/tor/control_auth_cookie
```

Your user must belong to the corresponding Tor group:

- `tor` on Arch Linux;
- `debian-tor` on Kali Linux.

The directory and cookie must also allow access for that group.

### Tor Fails to Start

```bash
sudo systemctl reset-failed tor
sudo tor -f /etc/tor/torrc --verify-config
sudo journalctl -u tor -n 50 --no-pager
```

### Check That Tor Is Listening

```bash
ss -tlnp | grep -E '9050|9051'
```

## Arch / Kali Summary

| Item | Arch Linux | Kali Linux |
|---|---|---|
| Installation | `sudo pacman -S tor torsocks openbsd-netcat` | `sudo apt install tor torsocks netcat-openbsd` |
| Tor group | `tor` | `debian-tor` |
| Service | `sudo systemctl enable --now tor` | `sudo systemctl enable --now tor` |
| SOCKS5 proxy | `127.0.0.1:9050` | `127.0.0.1:9050` |
| Configured ControlPort | `127.0.0.1:9051` | `127.0.0.1:9051` |
| Cookie | `/var/lib/tor/control_auth_cookie` | `/var/lib/tor/control_auth_cookie` |
| Script | Same | Same |

## Security

- Never expose `SocksPort` or `ControlPort` on `0.0.0.0`.
- Never run `chmod 777` on `/var/lib/tor` or the authentication cookie.
- `NEWNYM` applies to the Tor instance as a whole. It affects new connections from all applications using that instance, not just the command launched by `tor-rotate`.
- Do not use this script to scan, fuzz, or automate requests against a system without explicit authorization.