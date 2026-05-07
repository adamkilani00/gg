# Rapport TP – Réseaux IP
**Université de Strasbourg — Licence 2 Informatique**  
**Module : Réseaux IP — Année 2025-2026**

---

## Réseau de départ

Le réseau IPv4 de départ est : **192.168.64.0/18**

---

## Question 1 — Création de deux sous-réseaux d'au plus 2000 machines

### Raisonnement

On veut que chaque sous-réseau puisse accueillir **au plus 2000 machines**.

Le nombre d'hôtes utilisables dans un sous-réseau = **2ⁿ - 2** (on retire l'adresse réseau et le broadcast).

| Bits hôtes (n) | Adresses totales | Hôtes utilisables | Suffisant ? |
|:-:|:-:|:-:|:-:|
| 10 | 1024 | 1022 | ❌ Non |
| 11 | 2048 | 2046 | ✅ Oui |

Il faut **11 bits** pour les hôtes → le préfixe est **32 - 11 = /21**

### Sous-réseaux choisis

- **Sous-réseau A** : `192.168.64.0/21`
- **Sous-réseau B** : `192.168.72.0/21` (bloc suivant après A)

> Le réseau B commence à .72 car le bloc A couvre .64.0 → .71.255 (8 blocs de 256 adresses = 2048 adresses).

---

## Question 2 — Détails des sous-réseaux A et B

### Calcul du masque /21

Un masque /21 signifie 21 bits à 1 : `11111111.11111111.11111000.00000000` → **255.255.248.0**

### Sous-réseau A — 192.168.64.0/21

| Paramètre | Valeur |
|---|---|
| Adresse réseau | 192.168.64.0 |
| Masque (décimal pointé) | 255.255.248.0 |
| Première adresse IP utilisable | 192.168.64.1 |
| Dernière adresse IP utilisable | 192.168.71.254 |
| Adresse de broadcast | 192.168.71.255 |
| Nombre d'hôtes max | 2046 |

### Sous-réseau B — 192.168.72.0/21

| Paramètre | Valeur |
|---|---|
| Adresse réseau | 192.168.72.0 |
| Masque (décimal pointé) | 255.255.248.0 |
| Première adresse IP utilisable | 192.168.72.1 |
| Dernière adresse IP utilisable | 192.168.79.254 |
| Adresse de broadcast | 192.168.79.255 |
| Nombre d'hôtes max | 2046 |

---

## Question 3 — Topologie 1 (PC1 et PC2 reliés via un switch)

```
  PC1 ──────┐
            Switch
  PC2 ──────┘
```

### (a) Configuration de PC1 — sous-réseau A

**Commandes sur PC1 (Linux) :**

```bash
ip addr add 192.168.64.1/21 dev eth0
ip link set eth0 up
```

**Explication de chaque commande :**

| Commande | Rôle |
|---|---|
| `ip addr add 192.168.64.1/21 dev eth0` | Assigne l'adresse IP `192.168.64.1` avec le masque `/21` à l'interface `eth0` |
| `ip link set eth0 up` | Active (allume) l'interface réseau `eth0`. Sans cette commande, l'interface est désactivée et ne peut pas communiquer |

**Pourquoi /21 ?** Car PC1 doit être dans le sous-réseau A (192.168.64.0/21).

---

### (b) Configuration de PC2 — sous-réseau B avec l'erreur /19

**Commandes sur PC2 (Linux) :**

```bash
ip addr add 192.168.72.1/19 dev eth0
ip link set eth0 up
```

**Pourquoi c'est une erreur ?**

Avec un masque `/19`, PC2 calcule son réseau comme suit :
- 192.168.72.1 avec /19 → réseau = **192.168.64.0/19**
- Ce réseau couvre : 192.168.64.0 → 192.168.95.255

PC2 croit donc être dans le **même réseau** que PC1 (qui est en 192.168.64.1). Or en réalité, ils sont dans deux sous-réseaux distincts (/21). C'est l'erreur de configuration demandée.

---

### (c) Lancer Wireshark

**Commande :**

```bash
wireshark &
```

**Explication :**
- `wireshark` lance l'application d'analyse réseau
- `&` lance Wireshark **en arrière-plan** (le terminal reste disponible)

Dans Wireshark : sélectionner l'interface **eth0**, puis cliquer sur **Start** (bouton bleu ▶).

---

### (d) Test de connectivité PC1 → PC2

**Commande sur PC1 :**

```bash
ping 192.168.72.1
```

**Ce qui se passe :**

PC1 a le masque /21, donc il calcule : son réseau est **192.168.64.0/21** (couvre .64.0 → .71.255).

L'adresse de PC2 (192.168.72.1) est **en dehors** de ce réseau selon PC1.

PC1 cherche alors une **passerelle (routeur)** pour y acheminer le paquet. Aucune passerelle n'est configurée → le paquet ne peut pas partir.

**Résultat : le ping échoue.** On obtient des messages `Network unreachable` ou simplement aucune réponse.

**Dans Wireshark sur PC1 :** on voit des requêtes **ARP** de type "Who has \<gateway\>?" sans réponse — PC1 cherche un routeur inexistant.

---

### (e) Test de connectivité PC2 → PC1

**Commande sur PC2 :**

```bash
ping 192.168.64.1
```

**Ce qui se passe :**

PC2 a le masque /19, donc son réseau est **192.168.64.0/19** (couvre .64.0 → .95.255).

L'adresse de PC1 (192.168.64.1) est **dans ce réseau** selon PC2. PC2 envoie donc directement une requête **ARP** : "Qui a l'adresse 192.168.64.1 ?"

PC1 répond à l'ARP (il est bien sur le réseau physique), et reçoit le paquet ICMP de PC2.

Mais quand PC1 veut **répondre** à 192.168.72.1 (PC2), il voit que cette adresse est hors de son réseau /21, et cherche une passerelle → il n'en a pas → **la réponse ne part pas**.

**Résultat : ping asymétrique.** PC2 envoie des paquets, mais ne reçoit jamais de réponse. La communication est **à sens unique**.

**Dans Wireshark :** on voit les requêtes ICMP partir de PC2, les requêtes ARP réussir, mais aucun ICMP reply ne revient.

---

## Question 4 — Topologie 2 (avec 2 routeurs Cisco)

```
PC1 ── Routeur1 ──── Routeur2 ── PC2
```

Le lien entre les deux routeurs utilise le réseau **10.0.0.0/30** (lien point-à-point, 2 adresses utilisables).

### (a) Configuration de PC1 et PC2 (bons préfixes /21)

**PC1 :**

```bash
ip addr add 192.168.64.1/21 dev eth0
ip link set eth0 up
```

**PC2 :**

```bash
ip addr add 192.168.72.1/21 dev eth0
ip link set eth0 up
```

> Cette fois, les deux PC utilisent le **bon** masque /21.

---

### (b) Configuration des routeurs Cisco

#### Connexion à un routeur Cisco via câble console

```bash
# Trouver le port USB du câble console
ls /dev/ttyUSB*

# Se connecter au routeur
screen /dev/ttyUSB0 9600
```

**Explication :**
- `/dev/ttyUSB0` → port USB où est branché le câble RJ45→USB
- `9600` → vitesse de communication en bauds (valeur standard Cisco)
- Appuyer sur **Entrée** après connexion pour voir le prompt

**Pour quitter screen :** `Ctrl+A` puis `K` puis `Y`

#### Les modes Cisco — fondamental !

| Mode | Prompt | Utilité |
|---|---|---|
| User | `Router>` | Consultation uniquement |
| Privilégié | `Router#` | Diagnostics, commandes show |
| Configuration | `Router(config)#` | Modifier la configuration |

**Transitions entre modes :**

```
Router>  enable                → passe en mode privilégié
Router#  configure terminal    → passe en mode configuration
Router(config)#  exit          → revient en arrière
Router#  disable               → retour en mode user
```

---

#### Configuration de Routeur 1

```
Router1(config)# interface fastEthernet 0/0
Router1(config-if)# ip address 192.168.64.254 255.255.248.0
Router1(config-if)# no shutdown
Router1(config-if)# exit

Router1(config)# interface fastEthernet 0/1
Router1(config-if)# ip address 10.0.0.1 255.255.255.252
Router1(config-if)# no shutdown
Router1(config-if)# exit

Router1# write memory
```

**Explication commande par commande :**

| Commande | Rôle |
|---|---|
| `interface fastEthernet 0/0` | Entre dans la config de l'interface fa0/0 (côté PC1) |
| `ip address 192.168.64.254 255.255.248.0` | Donne l'IP 192.168.64.254 avec masque 255.255.248.0 à cette interface |
| `no shutdown` | **Active** l'interface (sur Cisco, toutes les interfaces sont éteintes par défaut !) |
| `exit` | Sort de la configuration de l'interface |
| `interface fastEthernet 0/1` | Entre dans la config de l'interface fa0/1 (côté Routeur 2) |
| `ip address 10.0.0.1 255.255.255.252` | Donne l'IP du lien inter-routeurs (/30 = 2 hôtes) |
| `write memory` | **Sauvegarde** la configuration en mémoire flash (sinon perdue au reboot !) |

**Pourquoi /30 pour le lien inter-routeurs ?** 2³⁰ = 4 adresses, dont 2 utilisables. Suffisant pour un lien point-à-point entre 2 équipements.

---

#### Configuration de Routeur 2

```
Router2(config)# interface fastEthernet 0/0
Router2(config-if)# ip address 192.168.72.254 255.255.248.0
Router2(config-if)# no shutdown
Router2(config-if)# exit

Router2(config)# interface fastEthernet 0/1
Router2(config-if)# ip address 10.0.0.2 255.255.255.252
Router2(config-if)# no shutdown
Router2(config-if)# exit

Router2# write memory
```

---

### (c) Route sur PC1 vers le sous-réseau B

**Commande sur PC1 :**

```bash
ip route add 192.168.72.0/21 via 192.168.64.254
```

**Explication :**

| Partie | Signification |
|---|---|
| `ip route add` | Ajoute une nouvelle route dans la table de routage |
| `192.168.72.0/21` | Réseau destination : le sous-réseau B |
| `via 192.168.64.254` | Passerelle à utiliser : Routeur 1 (son adresse côté PC1) |

PC1 sait désormais : "pour joindre le réseau B, passe par 192.168.64.254 (Routeur 1)."

---

### (d) Route sur PC2 vers le sous-réseau A

**Commande sur PC2 :**

```bash
ip route add 192.168.64.0/21 via 192.168.72.254
```

Même logique : PC2 passe par Routeur 2 (192.168.72.254) pour atteindre le réseau A.

---

### (e) Capture Wireshark depuis PC1 + ping vers PC2

**Commande :**

```bash
wireshark &
ping 192.168.72.1
```

**Ce qui se passe (analyse Wireshark) :**

1. PC1 regarde sa table de routage → destination 192.168.72.1 → passe par 192.168.64.254 (Routeur 1)
2. PC1 envoie une **requête ARP** : "Qui a 192.168.64.254 ?"
3. Routeur 1 répond à l'ARP avec son adresse MAC
4. PC1 envoie le **paquet ICMP Echo Request** vers Routeur 1
5. Routeur 1 reçoit le paquet et le transmet vers Routeur 2 (via 10.0.0.2)
6. Routeur 2 reçoit le paquet et doit le livrer à PC2 (192.168.72.1)
7. PC2 reçoit le ping et veut **répondre** à 192.168.64.1 (PC1)
8. PC2 a une route vers le réseau A via Routeur 2 → OK
9. **Mais Routeur 2 n'a pas de route de retour vers le réseau A !**
10. Routeur 2 ne sait pas comment acheminer la réponse → le paquet ICMP reply est perdu

**Résultat : le ping échoue** (aucune réponse reçue par PC1).

**Dans Wireshark :** on voit les ICMP Echo Requests partir de PC1, mais aucun Echo Reply ne revient.

---

### (f) Assurer la connectivité complète

Il manque les **routes sur les routeurs**. Il faut leur apprendre comment atteindre les réseaux distants.

#### Sur Routeur 1 (Cisco) — route vers le réseau B :

```
Router1(config)# ip route 192.168.72.0 255.255.248.0 10.0.0.2
Router1# write memory
```

**Explication :**
- `ip route` → ajoute une route statique
- `192.168.72.0 255.255.248.0` → réseau destination (réseau B) avec son masque
- `10.0.0.2` → passerelle : l'adresse de Routeur 2 sur le lien inter-routeurs

#### Sur Routeur 2 (Cisco) — route vers le réseau A :

```
Router2(config)# ip route 192.168.64.0 255.255.248.0 10.0.0.1
Router2# write memory
```

Même logique : Routeur 2 sait que pour joindre le réseau A, il passe par 10.0.0.1 (Routeur 1).

#### Le chemin complet est maintenant opérationnel :

```
PC1 → R1 → R2 → PC2   (aller)
PC2 → R2 → R1 → PC1   (retour)
```

**Ping réussi !** Dans Wireshark on voit les ICMP Echo Requests ET les ICMP Echo Replies.

---

## Résumé de toutes les commandes

### Commandes Linux (PC1 / PC2)

| Commande | Rôle |
|---|---|
| `ip addr add X.X.X.X/N dev eth0` | Assigne une adresse IP + masque à l'interface eth0 |
| `ip link set eth0 up` | Active l'interface réseau eth0 |
| `ip route add X.X.X.X/N via Y.Y.Y.Y` | Ajoute une route vers un réseau via une passerelle |
| `ping X.X.X.X` | Teste la connectivité vers une adresse IP |
| `wireshark &` | Lance Wireshark en arrière-plan |
| `ls /dev/ttyUSB*` | Liste les ports USB disponibles (pour câble console) |
| `screen /dev/ttyUSB0 9600` | Connexion console à un équipement réseau |

### Commandes Cisco (Routeurs)

| Commande | Rôle |
|---|---|
| `enable` | Passe en mode privilégié (Router#) |
| `configure terminal` | Passe en mode configuration (Router(config)#) |
| `interface fastEthernet 0/0` | Entre dans la config d'une interface |
| `ip address X.X.X.X M.M.M.M` | Assigne une IP + masque décimal pointé à l'interface |
| `no shutdown` | Active l'interface (désactivée par défaut sur Cisco) |
| `exit` | Sort du mode courant |
| `ip route X.X.X.X M.M.M.M Y.Y.Y.Y` | Ajoute une route statique |
| `show ip interface brief` | Affiche toutes les interfaces et leur état |
| `show ip route` | Affiche la table de routage |
| `write memory` | Sauvegarde la configuration (persiste au reboot) |

---

## Schéma récapitulatif de la Topologie 2

```
[PC1]                                              [PC2]
192.168.64.1/21                             192.168.72.1/21
     |                                               |
     | fa0/0                                   fa0/0 |
[Routeur 1]                               [Routeur 2]
192.168.64.254/21                       192.168.72.254/21
     | fa0/1                                   fa0/1 |
     |    10.0.0.1/30 ────────── 10.0.0.2/30        |
     └───────────────────────────────────────────────┘
                    Lien inter-routeurs
```

**Tables de routage finales :**

| Machine | Réseau destination | Passerelle |
|---|---|---|
| PC1 | 192.168.72.0/21 | 192.168.64.254 (R1) |
| PC2 | 192.168.64.0/21 | 192.168.72.254 (R2) |
| Routeur 1 | 192.168.72.0/21 | 10.0.0.2 (R2) |
| Routeur 2 | 192.168.64.0/21 | 10.0.0.1 (R1) |

---

*Rapport rédigé dans le cadre du TP noté — Réseaux IP L2 Informatique, Université de Strasbourg 2025-2026*
