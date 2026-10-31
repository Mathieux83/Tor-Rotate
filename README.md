# Tor + rotation automatique sur Arch Linux et Kali Linux

Ce guide installe et configure Tor sur **Arch Linux** ou **Kali Linux**, puis crée un script unique `tor-rotate` qui :

- lance une commande via Tor avec `torsocks` ;
- demande un nouveau circuit Tor au démarrage ;
- demande une nouvelle rotation toutes les 10 secondes pendant l’exécution ;
- retourne le code de sortie de la commande lancée.

## Installation sur Arch Linux

```bash
sudo pacman -Syu
sudo pacman -S tor torsocks openbsd-netcat
```

Active et démarre le service :

```bash
sudo systemctl enable --now tor
sudo systemctl status tor
```

## Installation sur Kali Linux

```bash
sudo apt update
sudo apt install tor torsocks netcat-openbsd
```

Active et démarre le service :

```bash
sudo systemctl enable --now tor
sudo systemctl status tor
```

## Configuration de `/etc/tor/torrc`

La configuration est identique sur Arch Linux et Kali Linux.

```bash
sudo nano /etc/tor/torrc
```

Ajoute ou vérifie ces lignes :

```conf
SocksPort 127.0.0.1:9050
SafeSocks 1
TestSocks 1

ControlPort 127.0.0.1:9051
CookieAuthentication 1
DataDirectoryGroupReadable 1
CookieAuthFileGroupReadable 1
```

Redémarre Tor :

```bash
sudo systemctl restart tor
sudo systemctl status tor
```

Tu dois voir dans les logs :

```text
Opening Socks listener on 127.0.0.1:9050
Opening Control listener on 127.0.0.1:9051
```

## Accès au cookie Tor

### Arch Linux

Ajoute ton utilisateur au groupe `tor` :

```bash
sudo usermod -aG tor $USER
```

### Kali Linux

Ajoute ton utilisateur au groupe `debian-tor` :

```bash
sudo usermod -aG debian-tor $USER
```

Dans les deux cas, reboot le pc et vérifie :

```bash
id
```

Tu dois voir le groupe Tor correspondant :

```text
Arch Linux : groups=...,tor
Kali Linux : groups=...,debian-tor
```

Le cookie se trouve ici :

```bash
/var/lib/tor/control_auth_cookie
```

Vérifie l’accès :

```bash
cat /var/lib/tor/control_auth_cookie >/dev/null && echo "Accès OK"
```

Si l’accès est refusé :

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

## Créer le script `tor-rotate` 

Le script est identique sur Arch Linux et Kali Linux.

**Ou simplement copie le du dossier vers ton ~/.local/bin/**

```bash
mkdir -p ~/.local/bin
nano ~/.local/bin/tor-rotate
```

Colle ce contenu :

```bash
#!/usr/bin/env bash
set -euo pipefail

COOKIE=/var/lib/tor/control_auth_cookie
HOST=127.0.0.1
PORT=9051
DELAY="${TOR_ROTATE_DELAY:-10}"

if [[ $# -eq 0 ]]; then
  echo "Usage : tor-rotate COMMANDE [arguments...]" >&2
  exit 1
fi

if (( DELAY < 10 )); then
  DELAY=10
fi

if [[ ! -r "$COOKIE" ]]; then
  echo "Erreur : cookie Tor non lisible : $COOKIE" >&2
  exit 1
fi

rotate() {
  local cookie_hex response
  cookie_hex=$(xxd -p "$COOKIE" | tr -d '\n')

  response=$(printf 'AUTHENTICATE %s\r\nSIGNAL NEWNYM\r\nQUIT\r\n' "$cookie_hex" \
    | timeout 5 nc "$HOST" "$PORT")

  if grep -q '^250 OK' <<<"$response"; then
    echo "[tor-rotate] $(date '+%H:%M:%S') — Nouveau circuit demandé."
  else
    echo "[tor-rotate] Erreur Tor :" >&2
    echo "$response" >&2
    return 1
  fi
}

rotate

echo "[tor-rotate] Lancement via Tor : $*"

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

Rends-le exécutable :

```bash
chmod +x ~/.local/bin/tor-rotate
```

Vérifie que `~/.local/bin` est dans ton `PATH` :

```bash
echo $PATH | grep -q "$HOME/.local/bin" || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Sur Kali avec Bash, utilise plutôt :

```bash
echo $PATH | grep -q "$HOME/.local/bin" || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## Utilisation

```bash
tor-rotate COMMANDE [arguments...]
```

La commande est automatiquement lancée avec `torsocks`, donc son trafic passe par Tor.

## Exemple 1 : tester que Tor fonctionne

```bash
tor-rotate curl [https://check.torproject.org/api/ip](https://check.torproject.org/api/ip)
```

Résultat attendu :

```json
{"IsTor":true,"IP":"x.x.x.x"}
```

## Exemple 2 : tester la rotation automatique

Cette commande effectue 6 requêtes espacées de 5 secondes, soit environ 30 secondes :

```bash
tor-rotate bash -c 'for i in {1..6}; do curl -s [https://check.torproject.org/api/ip](https://check.torproject.org/api/ip); echo; sleep 5; done'
```

Tu verras plusieurs messages :

```text
[tor-rotate] 19:30:07 — Nouveau circuit demandé.
[tor-rotate] 19:30:17 — Nouveau circuit demandé.
[tor-rotate] 19:30:27 — Nouveau circuit demandé.
```

Les IP peuvent changer, mais ce n’est pas garanti à chaque rotation : `NEWNYM` demande de nouveaux circuits, il ne garantit pas un nouveau relais de sortie.

## Exemple 3 : utiliser ffuf via Tor

```bash
tor-rotate ffuf -u http://cible/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

Tor ralentit fortement le fuzzing. Pour limiter la charge, réduis le nombre de threads :

```bash
tor-rotate ffuf -u http://cible/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -t 10
```

## Changer l’intervalle de rotation

Par défaut, la rotation est demandée toutes les 10 secondes :

```bash
TOR_ROTATE_DELAY=15 tor-rotate curl [https://check.torproject.org/api/ip](https://check.torproject.org/api/ip)
```

Ne descends pas sous 10 secondes : Tor limite les demandes `NEWNYM` trop rapprochées.

## Dépannage

### `nc: command not found`

```bash
# Arch Linux
sudo pacman -S openbsd-netcat

# Kali Linux
sudo apt install netcat-openbsd
```

### `cookie Tor non lisible`

Vérifie :

```bash
id
ls -ld /var/lib/tor
ls -l /var/lib/tor/control_auth_cookie
```

Ton utilisateur doit être dans le groupe Tor correspondant :

- `tor` sur Arch Linux ;
- `debian-tor` sur Kali Linux.

### Tor ne démarre plus

```bash
sudo systemctl reset-failed tor
sudo tor -f /etc/tor/torrc --verify-config
sudo journalctl -u tor -n 50 --no-pager
```

### Vérifier que Tor écoute

```bash
ss -tlnp | grep -E '9050|9051'
```

## Récapitulatif Arch / Kali

| Élément | Arch Linux | Kali Linux |
|---|---|---|
| Installation | `sudo pacman -S tor torsocks openbsd-netcat` | `sudo apt install tor torsocks netcat-openbsd` |
| Groupe Tor | `tor` | `debian-tor` |
| Service | `sudo systemctl enable --now tor` | `sudo systemctl enable --now tor` |
| Proxy SOCKS5 | `127.0.0.1:9050` | `127.0.0.1:9050` |
| ControlPort | `127.0.0.1:9051` | `127.0.0.1:9051` |
| Cookie | `/var/lib/tor/control_auth_cookie` | `/var/lib/tor/control_auth_cookie` |
| Script | Identique | Identique |

## Sécurité

- N’expose jamais `SocksPort` ou `ControlPort` sur `0.0.0.0`.
- Ne mets pas `chmod 777` sur `/var/lib/tor` ou sur le cookie.
- `NEWNYM` est global à Tor : il concerne les nouvelles connexions de toutes les applications utilisant ce Tor, pas uniquement la commande lancée par `tor-rotate`.
- N’utilise pas ce script pour scanner, fuzzé ou automatiser des requêtes contre un système sans autorisation explicite.