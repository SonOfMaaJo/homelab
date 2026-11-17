# Journal des évolutions

Les grandes étapes de la construction du homelab, et les décisions qui expliquent son état actuel.

## Chronologie

| Date | Évolution |
|---|---|
| Août 2026 | Premier service : transfert de fichiers (Gokapi) derrière un tunnel Cloudflare, secrets dans gopass. |
| 27 oct. 2026 | Le nœud Proxmox est remis en service sur le réseau. Accès distant par Tailscale. |
| 28 oct. 2026 | Le dépôt Git nu des secrets est remplacé par une forge Forgejo. |
| 30 oct. 2026 | Mise en place du socle : supervision (Uptime Kuma, ntfy, Homepage), exécuteur CI, Immich, proxy interne Caddy avec certificat wildcard, DNS AdGuard Home, Syncthing, Paperless-ngx. Mises à jour de sécurité automatiques sur tous les invités. Alertes sur le thin pool et sur les mises à jour de l'hôte. |
| 31 oct. 2026 | Chaîne de numérisation : un scan sur le téléphone arrive dans Paperless par Syncthing. |
| 5 nov. 2026 | Seconde copie des sauvegardes sur SSD USB, déclenchée au branchement. |
| 12 nov. 2026 | Jitsi Meet (accès par jeton), relais TURN, alerte sur le changement d'adresse IP publique. Le SSD de sauvegarde passe en chiffrement LUKS2. |
| 14 nov. 2026 | Règles d'isolation entre invités dans le pare-feu Proxmox. Sondes push pour ce que Kuma ne voit pas. Agent IA d'exploitation et sa passerelle. |
| 15 nov. 2026 | Second canal d'alerte par SMS (téléphone sentinelle). Première version de cette documentation. |
| 16 nov. 2026 | Essai de bout en bout de l'agent IA. Surveillance d'une télévision de la maison. |

## Décisions

**Un LAN plat plutôt que des VLAN.** Le matériel réseau de la maison (box du FAI, une seule interface sur le serveur) ne s'y prête pas. Le cloisonnement est fait là où il est possible : dans le pare-feu de l'hyperviseur, invité par invité.

**Des tunnels plutôt que des ports ouverts.** Le FAI ne permet pas de rediriger les ports web, et un tunnel sortant évite d'exposer l'adresse de la maison. Seul le média de la visioconférence, qui ne passe pas dans un tunnel HTTP, a justifié deux redirections.

**Un tunnel par invité public.** Un seul tunnel partagé aurait été plus simple, mais il aurait fait d'un invité le point de passage vers tous les autres.

**La forge est publique, pas derrière une authentification Cloudflare.** Une couche d'accès avait été placée devant Forgejo ; elle a été retirée le 30 oct. 2026. La forge se protège elle-même : inscriptions fermées, double authentification, dépôt des secrets privé (et de toute façon chiffré).

**La photothèque reste interne.** Immich n'est pas exposé publiquement : l'accès hors de la maison passe par le tailnet.

**Pas de mises à jour automatiques sur l'hyperviseur.** Les invités reçoivent leurs correctifs de sécurité seuls ; l'hôte est mis à jour à la main, après une alerte, pour qu'un redémarrage de noyau ne survienne jamais sans surveillance.

**Deux niveaux de sauvegarde, dont un hors ligne.** PBS est quotidien et versionné, mais il vit sur le même disque que ce qu'il sauvegarde. Le SSD USB, branché ponctuellement puis rangé ailleurs, couvre la panne du disque, le vol et l'erreur de manipulation.

**Un agent IA, mais sans accès libre.** L'intérêt est d'avoir un diagnostic rédigé au moment de la panne, et la relance des cas simples. Le risque est borné par une passerelle côté hôte plutôt que par la confiance dans le modèle (voir [Supervision et agent IA](supervision.md)).

## Pistes

- Observer l'agent IA sur une panne réelle, et ajuster la liste blanche en conséquence.
- Répéter une restauration complète à partir du SSD USB.
- Filtrer la sortie des invités les plus exposés.
- Redondance de disque, si le matériel évolue.
