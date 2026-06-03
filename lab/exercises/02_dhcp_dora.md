# Exercice 2 — DHCP : observer DORA paquet par paquet

**Durée estimée :** 45 min
**Objectif :** capturer un échange DHCP complet (Discover / Offer / Request /
ACK), identifier les options portées par chaque message, et comprendre
pourquoi DHCP utilise un broadcast L2 alors qu'IP n'est pas encore
configuré.

## Manipulation

Côté `dhcp-server`, démarrez une capture filtrée sur les ports DHCP (67/68)&nbsp;:

```bash
docker exec -it lab_dhcp_server tcpdump -i eth0 -nn -e -v port 67 or port 68
```

> Note&nbsp;: `-e` affiche les adresses MAC, indispensables pour comprendre
> le broadcast L2.

Côté `client`, déclenchez une nouvelle demande de bail&nbsp;:

```bash
docker exec lab_client bash -c "dhclient -r eth0 2>/dev/null; dhclient -v eth0"
```

Observez les **4 paquets** DORA dans la capture, puis arrêtez tcpdump (Ctrl+c).

Affichez aussi les journaux applicatifs du serveur&nbsp;:

```bash
docker logs --tail 40 lab_dhcp_server
```

## À rendre — répondez directement dans ce fichier

### 1. Tableau DORA

Complétez en vous appuyant sur **votre propre capture**&nbsp;:

| Étape | Émetteur (IP src) | Destinataire (IP dst) | MAC src / dst | Options DHCP notables |
|-------------|-------------------|----------------------|---------------|----------------------|
| 1. Discover | `0.0.0.0` | `255.255.255.255` | client / `ff:ff:ff:ff:ff:ff` | option 53 = 1, option 55 = liste params |
| 2. Offer | `172.20.0.1` | `255.255.255.255` | serveur / `ff:ff:ff:ff:ff:ff` | option 53 = 2, option 51 = 43200s, option 3 = 172.20.0.1 |
| 3. Request | `0.0.0.0` | `255.255.255.255` | client / `ff:ff:ff:ff:ff:ff` | option 53 = 3, option 54 = 172.20.0.1, option 50 = 172.20.0.2 |
| 4. ACK | `172.20.0.1` | `255.255.255.255` | serveur / `ff:ff:ff:ff:ff:ff` | option 53 = 5, option 51 = 43200s, option 1 = 255.255.0.0 |

### 2. Configuration finale du client

```bash
docker exec lab_client ip -4 addr show eth0
docker exec lab_client ip route
docker exec lab_client cat /etc/resolv.conf   # peut être vide si non géré par dhclient
```

Notez **l'IP attribuée, le masque, la passerelle, les DNS, la durée de bail**.

> 💬 **Votre réponse :**
>
> IP attribuée : 172.20.0.2 / 16 Masque : 255.255.0.0 Passerelle (GW) : 172.20.0.1 DNS : 8.8.8.8 Durée de bail : 43200 secondes (12h)

### 3. Questions de réflexion

**Question 1.** Pourquoi le client utilise-t-il **`0.0.0.0` comme IP
source** pour le Discover, alors que c'est une adresse non routable&nbsp;?
Que se passerait-il avec n'importe quelle autre adresse&nbsp;?

> 💬 **Votre réponse :**
>
>  Le client n'a pas encore d'IP donc il met 0.0.0.0 par défaut. Avec une autre adresse, ça créerait un conflit ou le serveur rejetterait la requête.

**Question 2.** Pourquoi le **Request** est-il **rediffusé en broadcast**
alors que le client connaît déjà l'IP du serveur après l'Offer&nbsp;?

> 💬 **Votre réponse :**
>
> Le client ne peut toujours pas faire d'unicast sans IP configurée. Le broadcast permet aussi d'informer les autres serveurs DHCP que leur offre est refusée.

**Question 3.** À quoi sert le **transaction ID (xid)** présent dans les
4 paquets&nbsp;? Que se passerait-il s'il était omis dans un réseau avec
plusieurs serveurs DHCP&nbsp;?

> 💬 **Votre réponse :**
>
> Le xid permet d'associer chaque réponse à la bonne requête. Sans lui, sur un réseau avec plusieurs serveurs DHCP, le client ne saurait plus quelle réponse lui est destinée.

**Question 4.** Que renvoie le serveur si vous demandez explicitement une
adresse hors du pool (essayez `dhclient -v -s 172.20.1.99 eth0`)&nbsp;?
Justifiez.

> 💬 **Votre réponse :**
>
> Le serveur renvoie un DHCPNAK car l'adresse demandée est hors de son pool. Il ne peut pas attribuer ce qu'il ne gère pas.

**Question 5.** La directive `dhcp-authoritative` est active sur notre
serveur. Quel est son effet **comportemental** sur les NAK&nbsp;?

> 💬 **Votre réponse :**
>
> dhcp-authoritative fait que le serveur envoie un NAK immédiat si l'adresse du client est inconnue, au lieu d'ignorer. Ça force le client à refaire un DORA complet rapidement.

### 4. Renouvellement de bail (T1/T2)

Le bail est de 12&nbsp;h, T1 (renouvellement) à 6&nbsp;h, T2 (rebind) à 10&nbsp;h30.
En **2-3 phrases**, décrivez la différence entre un renouvellement T1 et
un rebind T2 (destinataire du paquet, comportement attendu).

> 💬 **Votre réponse :**
>
>À T1, le client contacte son serveur en unicast pour renouveler. À T2, si pas de réponse, il broadcast vers n'importe quel serveur DHCP disponible pour tenter un rebind.
