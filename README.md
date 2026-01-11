# Indépendance du réseau local (LAN) avec MikroTik + AdGuard Home + Proxmox

Date : 2026-01-11
Routeur : MikroTik RouterOS 7.20.6
DNS local : AdGuard Home `192.168.1.3`
Home Assistant : VM Proxmox `101` (IP `192.168.1.80`)
Proxmox (PVE) : `192.168.1.201`

## Objectif

Garantir que **si la box Free (bridge) tombe / Internet est coupé (panne WAN)** :

- le **LAN continue de fonctionner** (DHCP/DNS/accès services locaux),
- Home Assistant reste accessible **en local** par IP et par nom,
- les équipements restent stables (pas de reboot routeur induit par la perte d’Internet),
- le temps (NTP) reste cohérent côté LAN (fallback).

> Hypothèse : **MikroTik (Wi‑Fi) + Proxmox** sont alimentés par UPS (≈15 min).

---

## Architecture finale

Clients (PC/tel/IoT)
→ DNS annoncé par DHCP : **MikroTik `192.168.1.1`**
→ MikroTik DNS (cache + serveur DNS LAN)
→ upstream unique : **AdGuard Home `192.168.1.3`**
→ noms locaux résolus *sans Internet* via **DNS static MikroTik** (`*.home`)

---

## Paramètres réseau (référence)

- LAN MikroTik : `192.168.1.1/24` sur `bridge`
- WAN MikroTik : `1-sfp-sfpplus` (DHCP client)
- AdGuard Home : `192.168.1.3`
- Home Assistant : `192.168.1.80`
- Proxmox PVE : `192.168.1.201`
- Domaine local retenu : `*.home` (ex: `ha.home`)

---

## Configuration MikroTik — Changements appliqués

### 1) Watchdog : éviter reboot quand Internet tombe

But : éviter qu’une perte WAN déclenche un reboot du routeur.

```routeros
/system watchdog set watch-address=none
/system watchdog print
```

---

### 2) DNS MikroTik : serveur DNS LAN + upstream AdGuard

But : les clients interrogent le routeur, qui forward vers AdGuard.

```routeros
/ip dns set allow-remote-requests=yes
/ip dns set servers=192.168.1.3
/ip dns print
```

---

### 3) DHCP : annoncer le DNS = MikroTik (192.168.1.1)

But : rendre la config client déterministe et indépendante.

```routeros
/ip dhcp-server network set [find where address="192.168.1.0/24"] dns-server=192.168.1.1
/ip dhcp-server network print detail
```

---

### 4) DNS static : noms locaux “offline” (sans dépendre d’Internet)

But : même sans Internet, résolution des services internes.

```routeros
/ip dns static add name=ha.home address=192.168.1.80
/ip dns static add name=pve.home address=192.168.1.201
/ip dns static add name=adguard.home address=192.168.1.3
/ip dns static print
```

---

### 5) Nettoyage : suppression d’un nom DNS invalide

But : retirer l’entrée “bad name” (`.dsplexinside.z-server.me`) qui peut perturber.

```routeros
/ip dns static remove 0
/ip dns static print detail where name=".dsplexinside.z-server.me"
```

---

### 6) Désactiver DNS/NTP fournis par le WAN (DHCP client WAN)

But : supprimer les `dynamic-servers` et garder un seul chemin DNS (AdGuard).

```routeros
/ip dhcp-client set 1 use-peer-dns=no use-peer-ntp=no
/ip dhcp-client print detail where interface="1-sfp-sfpplus"
/ip dns print
```

Attendu : `/ip dns print` → `dynamic-servers:` vide.

---

### 7) NTP : serveur local + fallback en cas de perte Internet

But : maintenir une heure cohérente sur le LAN (même si l’Internet tombe).

NTP client (déjà actif) :

- `0.fr.pool.ntp.org`
- `1.fr.pool.ntp.org`

NTP server (fallback local clock) :

```routeros
/system ntp server set use-local-clock=yes local-clock-stratum=10
/system ntp server print
```

---

### 8) Firewall NTP : LAN-only (UDP/123)

But : rendre explicite l’accept LAN et drop WAN pour NTP.

```routeros
/ip firewall filter add chain=input in-interface-list=LAN protocol=udp dst-port=123 action=accept comment="Allow NTP from LAN"
/ip firewall filter add chain=input in-interface-list=WAN protocol=udp dst-port=123 action=drop comment="Drop NTP from WAN"
```

Placement (les règles NTP ont été déplacées juste avant le drop `!LAN`) :

```routeros
/ip firewall filter move 26 destination=16
/ip firewall filter move 27 destination=16
/ip firewall filter print
```

Note : le routeur avait déjà une règle `drop in-interface-list=!LAN`, donc le WAN ne pouvait déjà pas atteindre les services input depuis l’extérieur. Les règles NTP rendent le tout lisible.

---

### 9) DHCP : allonger le lease-time (stabilité)

But : réduire les renouvellements DHCP, éviter l’effet “tout bouge” lors d’incidents.

```routeros
/ip dhcp-server set [find where name="defconf"] lease-time=1d
/ip dhcp-server print
```

---

## Configuration Proxmox — Vérification appliquée

Home Assistant VMID : `101`

But : HA redémarre automatiquement après reboot Proxmox.

```bash
qm config 101 | egrep -n '^(name:|onboot:|startup:|boot:|net0:|ipconfig0:)'
```

Attendu :

- `onboot: 1`
- `startup: order=1`

---

## Tests de validation

### Test SAFE (à distance, sans couper WAN)

Depuis MikroTik :

```routeros
:put [:resolve "ha.home"]
:put [:resolve "pve.home"]
:put [:resolve "adguard.home"]
ping 192.168.1.80 count=2
ping 192.168.1.3 count=2
```

Résultat attendu :

- résolutions OK
- pings HA + AdGuard OK

Note : `ping 192.168.1.201` (PVE) a timeout alors que l’ARP est reachable — cela indique généralement un filtrage ICMP côté PVE (ce n’est pas bloquant pour l’indépendance LAN).

---

### Test “preuve” (à faire quand tu es sur place)

Objectif : simuler “Internet down” sans toucher à la box.

1) Couper WAN MikroTik (≈30s) :

```routeros
/interface disable 1-sfp-sfpplus
```

2) Depuis un appareil Wi‑Fi LAN :

- ouvrir `http://ha.home:8123`  (attendu : OK)
- ouvrir `http://adguard.home`  (attendu : OK)
- ouvrir `https://pve.home:8006` (attendu : OK — warning TLS possible)

3) Remettre WAN :

```routeros
/interface enable 1-sfp-sfpplus
```

Si ces 3 accès fonctionnent pendant WAN down, l’indépendance LAN est confirmée.

---

## Ce qui restera KO (normal) quand Internet est down

- intégrations cloud (Google/Alexa, Tuya cloud, etc.)
- météo/API externes
- accès HA depuis l’extérieur sans VPN (Tailscale, etc.)
- tout service qui nécessite l’Internet pour fonctionner

---

## Rollback (si nécessaire)

- DHCP DNS (remettre AdGuard direct) :

  ```routeros
  /ip dhcp-server network set [find where address="192.168.1.0/24"] dns-server=192.168.1.3
  ```
- Réactiver DNS WAN (non recommandé pour “déterministe”) :

  ```routeros
  /ip dhcp-client set 1 use-peer-dns=yes use-peer-ntp=yes
  ```
- Retirer les statics `.home` :

  ```routeros
  /ip dns static remove [find where name="ha.home"]
  /ip dns static remove [find where name="pve.home"]
  /ip dns static remove [find where name="adguard.home"]
  ```
- Lease-time DHCP :

  ```routeros
  /ip dhcp-server set [find where name="defconf"] lease-time=10m
  ```

---

## Annexes — Indices utiles

- PVE ne répond pas au ping depuis MikroTik :
  - ARP `reachable` = connectivité L2 OK.
  - timeout ping = ICMP bloqué / firewall PVE / règle réseau.
  - L’UI web `https://pve.home:8006` peut rester OK malgré ping KO.
