# Réseau

## Topologie

Le homelab est volontairement plat : un seul LAN (un `/24` privé), un seul pont Proxmox (`vmbr0`) relié à l'unique interface physique. Il n'y a pas de VLAN ; le cloisonnement entre invités est assuré par le pare-feu Proxmox (voir [Sécurité](securite.md)).

```mermaid
flowchart LR
    box[Box FAI<br/>passerelle]
    subgraph hote["Hôte Proxmox"]
        br[vmbr0]
    end
    box --- br
    br --- g1[12 conteneurs LXC]
    br --- g2[1 VM]
    br -.- ts[Tailscale<br/>routeur de sous-réseau]
```

## Adressage

- Chaque invité a une **adresse statique** sur le LAN, hors de la plage DHCP de la box.
- Les adresses exactes ne sont volontairement pas publiées ici.
- Les invités sont référencés dans le pare-feu par un groupe d'adresses (`guests`), ce qui permet d'écrire des règles « tout invité → … » sans les énumérer.

## Trois chemins d'accès

### 1. Public : tunnels Cloudflare

Le FAI ne permet pas de rediriger les ports bas (80, 443). Les services publics sont donc exposés par des **tunnels Cloudflare** : un démon `cloudflared` dans chaque invité concerné ouvre une connexion *sortante* vers Cloudflare, qui termine le TLS et relaie les requêtes. Aucun port n'est ouvert sur la box pour le web.

Chaque invité public a son propre tunnel, ce qui évite qu'un invité compromis serve de rebond vers les autres.

| Nom public | Invité | Service |
|---|---|---|
| `vnaoussi-djoumessi.com` | VM 200 | Portfolio (nginx) |
| `files.vnaoussi-djoumessi.com` | CT 101 | Gokapi |
| `git.vnaoussi-djoumessi.com` | CT 103 | Forgejo (HTTPS uniquement) |
| `cms-auth.vnaoussi-djoumessi.com` | CT 103 | Proxy OAuth de Decap CMS |
| `ntfy.vnaoussi-djoumessi.com` | CT 105 | ntfy (notifications sur téléphone) |
| `meet.vnaoussi-djoumessi.com` | CT 112 | Jitsi Meet (interface web) |

**Exception Jitsi.** Les flux audio/vidéo ne passent pas par un tunnel HTTP. Deux ports sont redirigés sur la box vers le CT 112 : `10000/udp` (pont vidéo) et `443/tcp` (coturn, repli TURN sur TLS pour les réseaux restrictifs).

### 2. Interne : reverse proxy Caddy

Les services privés sont joignables en HTTPS sous `*.home.vnaoussi-djoumessi.com` via Caddy (CT 108).

- Les enregistrements DNS pointent vers l'adresse privée du proxy : ces noms ne répondent que depuis le LAN ou le tailnet.
- Le certificat est un **wildcard Let's Encrypt** obtenu par défi DNS (API Cloudflare). Aucun service n'a besoin d'être joignable depuis Internet pour obtenir un certificat valide.

| Nom interne | Cible |
|---|---|
| `accueil.home…` | Homepage (CT 105) |
| `kuma.home…` | Uptime Kuma (CT 105) |
| `photos.home…` | Immich (CT 107) |
| `adguard.home…` | AdGuard Home (CT 109) |
| `syncthing.home…` | Syncthing (CT 110) |
| `papiers.home…` | Paperless-ngx (CT 111) |
| `pve.home…` | Interface Proxmox VE (hôte) |
| `pbs.home…` | Interface Proxmox Backup Server (CT 104) |

### 3. Distant : Tailscale

L'hôte Proxmox est membre d'un tailnet et **annonce le sous-réseau du LAN** (routeur de sous-réseau). Depuis un ordinateur ou un téléphone connecté au tailnet, tous les services internes sont accessibles comme à la maison, y compris les noms `*.home`.

L'administration (SSH, interfaces Proxmox et PBS) n'est possible que depuis le LAN ou le tailnet.

Le tailnet sert aussi dans l'autre sens : l'hôte y joint un téléphone qui envoie les alertes par SMS (voir [Supervision](supervision.md)).

### Chemins SSH

| Cible | Chemin |
|---|---|
| Hôte Proxmox | Direct, par le tailnet ou le LAN |
| Conteneurs | Depuis l'hôte (`pct exec`), ou en SSH depuis le LAN |
| VM `debianS` | SSH sur un port non standard, par rebond sur l'hôte |
| Forge (Git sur SSH) | Port dédié de Forgejo, par rebond sur l'hôte depuis l'extérieur |

## Qui parle à qui

Le pare-feu refuse par défaut tout trafic d'un invité vers un autre. Le tableau liste les seules exceptions, c'est-à-dire tous les flux internes qui existent.

| Destination | Port | Depuis la supervision (105) | Depuis le proxy (108) | Depuis un autre invité | Depuis le LAN / tailnet |
|---|---|:-:|:-:|:-:|:-:|
| Hôte — interface Proxmox | 8006 | ✔ | ✔ | — | ✔ |
| Hôte — passerelle de l'agent (SSH) | 22 | — | — | 113 seulement | ✔ |
| Hôte — relais SMS | 8787 | ✔ | — | — | — |
| 101 `files-transfer` | — | ping | — | — | SSH |
| 103 `git-server` — Forgejo web | 3000 | ✔ | — | 106 (CI) | ✔ |
| 103 `git-server` — Git sur SSH | 2222 | ✔ | — | — | ✔ |
| 104 `pbs` | 8007 | ✔ | ✔ | — | ✔ |
| 105 `monitoring` — Homepage, Kuma | 3000, 3001 | — | ✔ | — | ✔ |
| 105 `monitoring` — ntfy | 8080 | — | — | tous (jeton requis) | ✔ |
| 106 `ci-runner` | — | ping | — | — | SSH |
| 107 `immich` | 2283 | ✔ | ✔ | — | ✔ |
| 108 `proxy` | 80, 443 | ✔ (443) | — | — | ✔ |
| 109 `adguard` — DNS | 53 | ✔ | ✔ | tous | ✔ |
| 109 `adguard` — interface | 80 | ✔ | ✔ | — | ✔ |
| 110 `syncthing` | 8384, 22000 | ✔ (8384) | ✔ (8384) | — | ✔ |
| 111 `paperless` | 8000 | ✔ | ✔ | — | ✔ |
| 112 `jitsi` — web | 8000 | ✔ | — | — | ✔ |
| 112 `jitsi` — média | 443, 10000/udp | — | — | — | ✔ et **Internet** (redirection de la box) |
| 113 `ops-agent` — webhook | 8787 | ✔ | — | — | — |
| 200 `debianS` — SSH, web local | port SSH, 80 | ✔ (SSH) | — | — | ✔ |

À lire ainsi :

- les services publics (101, 103, 112, 200) ne sont **pas** derrière le proxy interne : ils sortent par leur propre tunnel ;
- le CT 106 et le CT 101 n'offrent aucun port aux autres invités ;
- le webhook de l'agent IA et le relais SMS ne sont joignables **que** par la supervision, pas même depuis le LAN ;
- en sortie, les invités ne sont pas filtrés (mises à jour, tunnels, API externes).

## DNS

AdGuard Home (CT 109) sert de résolveur filtrant pour le LAN et pour les invités. C'est l'un des premiers conteneurs à démarrer (ordre 1, avec PBS).

| Réglage | Valeur |
|---|---|
| Résolveurs amont | DNS-over-HTTPS, Cloudflare et Quad9 interrogés **en parallèle** (la réponse la plus rapide gagne) |
| Secours | Résolveurs classiques des deux mêmes fournisseurs, si le DoH est injoignable |
| Validation | DNSSEC activé |
| Cache | Cache optimiste : une réponse expirée est servie tout de suite, puis rafraîchie en arrière-plan |
| Filtrage | AdGuard DNS filter, HaGeZi Pro |
| Réécritures | `*.home.…` vers le proxy interne ; le nom d'administration de la box vers la box |
| Conservation | Journal des requêtes 7 jours, statistiques 30 jours |

La réécriture `*.home` rend les noms internes indépendants d'Internet : même connexion coupée, `photos.home…` continue de résoudre et de fonctionner à la maison. Le certificat, lui, se renouvelle par défi DNS et n'a besoin d'Internet qu'au moment du renouvellement.

## Ordre de démarrage

| Ordre | Invités |
|---|---|
| 1 | `adguard`, `pbs` |
| 2 | `proxy`, `monitoring` |
| 3 | `ci-runner`, `syncthing` |
| 4 | `immich`, `paperless`, `jitsi`, `ops-agent` |
| — | `files-transfer`, `git-server`, `debianS` (sans ordre imposé) |
