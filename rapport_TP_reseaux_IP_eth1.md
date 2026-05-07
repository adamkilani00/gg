# Rapport TP – Réseaux IP (version interface eth1)
**Université de Strasbourg — Licence 2 Informatique**  
**Module : Réseaux IP — Année 2025-2026**

---

## Pourquoi eth1 et pas eth0 ?

Sur une machine Linux, les interfaces réseau s'appellent `eth0`, `eth1`, `eth2`, etc.  
Le chiffre correspond à **l'ordre de détection** de la carte réseau par le système.

- `eth0` = première carte réseau (souvent utilisée pour autre chose, ex: accès internet ou gestion)
- `eth1` = deuxième carte réseau = **celle branchée au switch ou au routeur pour ce TP**

> Si ton PC a plusieurs ports réseau (ce qui est courant en salle de TP), il se peut que `eth0` soit déjà utilisée par le réseau de la fac, et que ton câble de TP soit branché sur `eth1`.

### Comment savoir quelle interface utiliser ?

```bash
ip link show
```

Cette commande **liste toutes les interfaces** disponibles. Tu verras quelque chose comme :

```
1: lo: <LOOPBACK> ...
2: eth0: <BROADCAST,MULTICAST,UP> ...
3: eth1: <BROADCAST,MULTICAST> ...
```

Tu peux aussi regarder quelle interface est branchée (câble connecté) :

```bash
ip link show eth1
```

Si tu vois `state UP` → le câble est branché sur eth1. Si tu vois `state DOWN` → pas de câble.

**Autre commande utile :**

```bash
ethtool eth1
```

Cherche la ligne `Link detected: yes` → ça confirme que le câble est bien branché sur eth1.

---

## Réseau de départ

Le réseau IPv4 de départ est : **192.168.64.0/18**

---

## Question 1 — Création de deux sous-réseaux d'au plus 2000 machines

### Raisonnement

| Bits hôtes (n) | Hôtes utilisables (2ⁿ - 2) | Suffisant ? |
|:-:|:-:|:-:|
| 10 | 1022 | ❌ Non |
| 11 | 2046 | ✅ Oui |

Il faut **11 bits** pour les hôtes → préfixe = **32 - 11 = /21**

### Sous-réseaux choisis

- **Sous-réseau A** : `192.168.64.0/21`
- **Sous-réseau B** : `192.168.72.0/21`

---

## Question 2 — Détails des sous-réseaux A et B

Masque /21 = **255.255.248.0**

### Sous-réseau A — 192.168.64.0/21

| Paramètre | Valeur |
|---|---|
| Adresse réseau | 192.168.64.0 |
| Masque | 255.255.248.0 |
| Première adresse IP | 192.168.64.1 |
| Dernière adresse IP | 192.168.71.254 |
| Broadcast | 192.168.71.255 |

### Sous-réseau B — 192.168.72.0/21

| Paramètre | Valeur |
|---|---|
| Adresse réseau | 192.168.72.0 |
| Masque | 255.255.248.0 |
| Première adresse IP | 192.168.72.1 |
| Dernière adresse IP | 192.168.79.254 |
| Broadcast | 192.168.79.255 |

---

## Question 3 — Topologie 1 (PC1 et PC2 reliés via un switch)

```
  PC1 ──────┐
            Switch
  PC2 ──────┘
```

### (a) Configuration de PC1 — sous-réseau A sur eth1

**Commandes sur PC1 :**

```bash
ip addr add 192.168.64.1/21 dev eth1
ip link set eth1 up
```

**Explication :**

| Commande | Rôle |
|---|---|
| `ip addr add 192.168.64.1/21 dev eth1` | Assigne l'IP 192.168.64.1 avec masque /21 à l'interface **eth1** |
| `ip link set eth1 up` | Active l'interface eth1 (sans ça, elle est éteinte et ne communique pas) |

> 💡 La seule différence avec eth0 : on écrit **eth1** partout à la place de **eth0**. La logique est exactement la même.

**Vérifier que c'est bien configuré :**

```bash
ip addr show eth1
```

Tu dois voir apparaître `192.168.64.1/21` dans la sortie.

---

### (b) Configuration de PC2 — sous-réseau B avec l'erreur /19

**Commandes sur PC2 :**

```bash
ip addr add 192.168.72.1/19 dev eth1
ip link set eth1 up
```

**Pourquoi c'est une erreur ?**  
Avec /19, PC2 croit que son réseau est 192.168.64.0/19 (couvre .64.0 → .95.255), donc il pense être dans le **même réseau** que PC1. C'est l'erreur volontaire du TP.

---

### (c) Lancer Wireshark sur eth1

```bash
wireshark &
```

Dans Wireshark : sélectionner l'interface **eth1** (et non eth0 !) puis cliquer sur ▶ Start.

> ⚠️ Si tu sélectionnes eth0 par erreur, tu ne verras aucun trafic lié au TP !

---

### 📸 CAPTURE D'ÉCRAN WIRESHARK — Ce qu'il faut faire

#### Étape 1 — Lancer la capture AVANT le ping

```bash
# Dans un terminal, lancer Wireshark
wireshark &
# Sélectionner eth1 → Start
```

#### Étape 2 — Faire le ping dans un autre terminal

```bash
ping 192.168.72.1
# Attendre 4-5 paquets puis Ctrl+C pour arrêter
```

#### Étape 3 — Arrêter la capture dans Wireshark

Cliquer sur le bouton **Stop** (carré rouge ■).

#### Étape 4 — Nettoyer le trafic parasite

Le sujet demande des traces **sans trafic parasite**. Il faut filtrer pour ne garder que ce qui est utile.

**Appliquer ce filtre dans la barre de filtre Wireshark :**

```
arp or icmp
```

Ce filtre affiche uniquement :
- **ARP** : les requêtes pour trouver les adresses MAC
- **ICMP** : les paquets ping (Echo Request / Echo Reply)

> Tout le reste (DNS, mDNS, LLMNR, etc.) sera caché → traces propres !

#### Étape 5 — Sauvegarder la trace filtrée

```
File → Export Specified Packets → cocher "Displayed" → Save
```

Nommer le fichier : `capture_Q3d_PC1_vers_PC2.pcapng`

---

### (d) Test de connectivité PC1 → PC2

**Commande sur PC1 :**

```bash
ping 192.168.72.1
```

**Ce qui se passe :**  
PC1 (masque /21) voit 192.168.72.1 comme hors réseau → cherche une passerelle → il n'y en a pas → **ping échoue**.

**Dans Wireshark :** requêtes ARP sans réponse (PC1 cherche un routeur inexistant).

---

### (e) Test de connectivité PC2 → PC1

**Commande sur PC2 :**

```bash
ping 192.168.64.1
```

**Ce qui se passe :**  
PC2 (masque /19) voit 192.168.64.1 comme dans son réseau → envoie ARP directement → PC1 répond à l'ARP → mais PC1 ne peut pas répondre au ping (192.168.72.1 est hors de son /21).

**Résultat : communication asymétrique** — les paquets partent mais aucune réponse ne revient.

---

## Question 4 — Topologie 2 (avec 2 routeurs Cisco)

```
PC1 ── Routeur1 ──── Routeur2 ── PC2
```

### (a) Configuration de PC1 et PC2 sur eth1 (bons préfixes /21)

**PC1 :**

```bash
ip addr add 192.168.64.1/21 dev eth1
ip link set eth1 up
```

**PC2 :**

```bash
ip addr add 192.168.72.1/21 dev eth1
ip link set eth1 up
```

---

### (b) Configuration des routeurs Cisco

#### Connexion au routeur via câble console

```bash
ls /dev/ttyUSB*        # trouver le port
screen /dev/ttyUSB0 9600
```

Appuyer sur **Entrée** → tu vois `Router>`

**Quitter screen :** `Ctrl+A` puis `K` puis `Y`

#### Les 3 modes Cisco

| Mode | Prompt | Accès |
|---|---|---|
| User | `Router>` | Par défaut |
| Privilégié | `Router#` | `enable` |
| Configuration | `Router(config)#` | `configure terminal` |

---

#### Routeur 1 — Configuration complète

```
Router> enable
Router# configure terminal

Router(config)# interface fastEthernet 0/0
Router(config-if)# ip address 192.168.64.254 255.255.248.0
Router(config-if)# no shutdown
Router(config-if)# exit

Router(config)# interface fastEthernet 0/1
Router(config-if)# ip address 10.0.0.1 255.255.255.252
Router(config-if)# no shutdown
Router(config-if)# exit

Router(config)# exit
Router# write memory
```

---

#### Routeur 2 — Configuration complète

```
Router> enable
Router# configure terminal

Router(config)# interface fastEthernet 0/0
Router(config-if)# ip address 192.168.72.254 255.255.248.0
Router(config-if)# no shutdown
Router(config-if)# exit

Router(config)# interface fastEthernet 0/1
Router(config-if)# ip address 10.0.0.2 255.255.255.252
Router(config-if)# no shutdown
Router(config-if)# exit

Router(config)# exit
Router# write memory
```

---

### 📄 FICHIER DE CONFIGURATION DES ROUTEURS — Comment le récupérer

Le sujet demande de **joindre les fichiers de configuration** des routeurs. Voici comment les obtenir :

#### Étape 1 — Afficher la config sur le routeur

```
Router# show running-config
```

Cette commande affiche **toute la configuration active** du routeur. Tu verras défiler les interfaces, les routes, etc.

#### Étape 2 — Copier le texte

Dans ton terminal (screen), **sélectionner tout le texte** qui s'affiche avec la souris et copier.

#### Étape 3 — Coller dans un fichier texte

Sur ton PC Linux, ouvre un éditeur :

```bash
nano config_routeur1.txt
```

Colle le contenu (`Ctrl+Shift+V` dans le terminal), puis sauvegarde (`Ctrl+X` → `Y` → Entrée).

#### Ce que le fichier doit contenir (exemple pour Routeur 1)

```
!
interface FastEthernet0/0
 ip address 192.168.64.254 255.255.248.0
 no shutdown
!
interface FastEthernet0/1
 ip address 10.0.0.1 255.255.255.252
 no shutdown
!
ip route 192.168.72.0 255.255.248.0 10.0.0.2
!
end
```

> Nommer les fichiers : `config_routeur1.txt` et `config_routeur2.txt`

---

### (c) Route sur PC1 vers le sous-réseau B

```bash
ip route add 192.168.72.0/21 via 192.168.64.254
```

**Explication :**

| Partie | Signification |
|---|---|
| `ip route add` | Ajoute une route dans la table de routage |
| `192.168.72.0/21` | Réseau destination = sous-réseau B |
| `via 192.168.64.254` | Passerelle = Routeur 1 (adresse côté PC1) |

**Vérifier la table de routage :**

```bash
ip route show
```

---

### (d) Route sur PC2 vers le sous-réseau A

```bash
ip route add 192.168.64.0/21 via 192.168.72.254
```

---

### (e) Capture Wireshark depuis PC1 + ping vers PC2

#### Lancer la capture sur eth1

```bash
wireshark &
# Sélectionner eth1 → Start
```

#### Faire le ping

```bash
ping 192.168.72.1
```

**Ce qui se passe (analyse) :**

1. PC1 consulte sa route → passe par 192.168.64.254 (Routeur 1)
2. PC1 envoie une requête **ARP** : "Qui a 192.168.64.254 ?"
3. Routeur 1 répond → PC1 obtient son adresse MAC
4. PC1 envoie le **paquet ICMP** vers Routeur 1
5. Routeur 1 transmet à Routeur 2 (via 10.0.0.2)
6. Routeur 2 reçoit mais **n'a pas de route de retour** vers 192.168.64.0/21
7. La réponse ICMP est perdue → **ping échoue**

**Dans Wireshark :** ICMP Echo Requests présents, mais aucun Echo Reply.

#### Filtre Wireshark à appliquer

```
arp or icmp
```

Sauvegarder : `capture_Q4e_PC1_vers_PC2.pcapng`

---

### (f) Assurer la connectivité complète

Ajouter les routes sur les deux routeurs :

#### Routeur 1

```
Router1(config)# ip route 192.168.72.0 255.255.248.0 10.0.0.2
Router1# write memory
```

#### Routeur 2

```
Router2(config)# ip route 192.168.64.0 255.255.248.0 10.0.0.1
Router2# write memory
```

#### Tester à nouveau

```bash
ping 192.168.72.1
```

**Résultat attendu :**

```
PING 192.168.72.1: 64 bytes data
64 bytes from 192.168.72.1: seq=0 ttl=62 time=2.4 ms
64 bytes from 192.168.72.1: seq=1 ttl=62 time=1.8 ms
```

**Dans Wireshark :** on voit maintenant les **Echo Request ET les Echo Reply** → connectivité bidirectionnelle confirmée.

Sauvegarder : `capture_Q4f_ping_OK.pcapng`

---

## Guide récapitulatif — Captures Wireshark à rendre

| Capture | Quand la faire | Filtre | Nom du fichier |
|---|---|---|---|
| Q3(d) — PC1→PC2 échec | Après ping depuis PC1 (topo 1) | `arp or icmp` | `capture_Q3d.pcapng` |
| Q3(e) — PC2→PC1 asymétrique | Après ping depuis PC2 (topo 1) | `arp or icmp` | `capture_Q3e.pcapng` |
| Q4(e) — PC1→PC2 sans routes routeurs | Après ping depuis PC1 (topo 2, avant Q4f) | `arp or icmp` | `capture_Q4e.pcapng` |
| Q4(f) — PC1→PC2 fonctionnel | Après ajout des routes sur routeurs | `arp or icmp` | `capture_Q4f.pcapng` |

---

## Résumé de toutes les commandes (version eth1)

### Commandes Linux

| Commande | Rôle |
|---|---|
| `ip link show` | Liste toutes les interfaces réseau disponibles |
| `ip link show eth1` | Vérifie l'état de l'interface eth1 (UP/DOWN) |
| `ip addr add X.X.X.X/N dev eth1` | Assigne une IP à l'interface eth1 |
| `ip link set eth1 up` | Active l'interface eth1 |
| `ip addr show eth1` | Vérifie la configuration IP de eth1 |
| `ip route add X.X.X.X/N via Y.Y.Y.Y` | Ajoute une route statique |
| `ip route show` | Affiche la table de routage |
| `ping X.X.X.X` | Teste la connectivité |
| `wireshark &` | Lance Wireshark en arrière-plan |

### Commandes Cisco

| Commande | Rôle |
|---|---|
| `enable` | Mode privilégié |
| `configure terminal` | Mode configuration |
| `interface fastEthernet 0/0` | Entre dans la config d'une interface |
| `ip address X.X.X.X M.M.M.M` | Assigne une IP + masque |
| `no shutdown` | Active l'interface |
| `exit` | Sort du mode courant |
| `ip route X.X.X.X M.M.M.M Y.Y.Y.Y` | Route statique |
| `show running-config` | Affiche la configuration complète (à copier pour le rendu) |
| `show ip interface brief` | État rapide de toutes les interfaces |
| `show ip route` | Table de routage |
| `write memory` | Sauvegarde la config |

---

## Schéma final — Topologie 2

```
[PC1]                                              [PC2]
192.168.64.1/21                             192.168.72.1/21
  (eth1)                                          (eth1)
     |                                               |
     | fa0/0                                   fa0/0 |
[Routeur 1]                               [Routeur 2]
192.168.64.254/21                       192.168.72.254/21
     | fa0/1                                   fa0/1 |
  10.0.0.1/30 ─────────────────────── 10.0.0.2/30
```

---

## Checklist finale avant de rendre le TP

- [ ] Rapport PDF avec toutes les questions numérotées
- [ ] Captures d'écran Wireshark intégrées dans le rapport
- [ ] 4 fichiers `.pcapng` propres (filtrés `arp or icmp`)
- [ ] `config_routeur1.txt` (issu de `show running-config`)
- [ ] `config_routeur2.txt` (issu de `show running-config`)
- [ ] Archive `.tar.gz` contenant tout
- [ ] Déposé sur Moodle avant le **dimanche 10/05 à 20h**

### Créer l'archive tar.gz

```bash
tar -czvf rendu_TP_reseaux.tar.gz rapport.pdf capture_Q3d.pcapng capture_Q3e.pcapng capture_Q4e.pcapng capture_Q4f.pcapng config_routeur1.txt config_routeur2.txt
```

**Explication :**

| Option | Rôle |
|---|---|
| `-c` | Créer une archive |
| `-z` | Compresser avec gzip (.gz) |
| `-v` | Verbose : affiche les fichiers ajoutés |
| `-f` | Spécifie le nom du fichier de sortie |

---

*Rapport rédigé dans le cadre du TP noté — Réseaux IP L2 Informatique, Université de Strasbourg 2025-2026*
