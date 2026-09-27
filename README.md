# 🏢 Campus Multi-VLAN avec Redondance de Liens

**Niveau :** Intermédiaire
**Outils :** Cisco Packet Tracer
**Thèmes couverts :** VLAN · Trunking 802.1Q · STP Rapid-PVST+ · EtherChannel (LACP) · DHCP (relay) · Routage inter-VLAN (SVI)

---

## 📋 Contexte / Scénario

Une PME répartie sur deux étages doit segmenter son trafic par service (**Direction, Comptabilité, Ventes, Invités**) tout en garantissant une continuité de service si un lien entre switches tombe en panne.

## 🎯 Objectifs pédagogiques

- Créer et nommer 3 VLANs (10-Direction, 20-Compta, 99-Invités) sur deux switches d'accès et un switch de distribution L3.
- Configurer des trunks 802.1Q en n'autorisant que les VLANs nécessaires (VLAN pruning).
- Regrouper deux liens entre SW-DIST et SW-ACCES1 en EtherChannel LACP (Po1) pour la bande passante et la redondance.
- Activer Rapid-PVST+ et fixer le switch de distribution comme pont racine (priorité) pour tous les VLANs.
- Créer les interfaces virtuelles (SVI) sur le switch L3 pour le routage inter-VLAN.
- Configurer un service DHCP centralisé sur le switch L3 avec `ip helper-address` ou relais si le serveur est distant.

## 🗺️ Schéma d'architecture

![Topologie réseau Packet Tracer](01_topologie_reseau_packet_tracer.png)

## 🔢 Plan d'adressage de base

| Segment / rôle       | Réseau            | SVI (passerelle) |
|-----------------------|-------------------|-------------------|
| VLAN 10 - Direction    | 192.168.10.0/24   | .1 |
| VLAN 20 - Comptabilité | 192.168.20.0/24   | .1 |
| VLAN 99 - Invités      | 192.168.99.0/24   | .1 |
| VLAN 1 - Management    | 192.168.1.0/24    | .1 |

---

## 🧭 Cahier des tâches — Preuves en capture

### 1️⃣ Créer les VLANs 10, 20, 99 et 1 (management) sur les 3 switches, avec des noms explicites

![Show VLAN brief sur les 3 switches](02_switches_show_vlan_brief.png)

`SH VLAN` exécuté sur **Switch0**, **Multilayer Switch0** et **Switch1** : les VLANs 10 (DIRECTION), 20 (COMPTA), 30 (VENTES) et 99 (MANAGEMENT) apparaissent bien nommés et actifs, avec la répartition des ports par VLAN sur chaque équipement.

![Vérification croisée VTP + VLAN brief](12_switches_vtp_status_and_vlan_brief.png)

Second passage de vérification confirmant la cohérence des VLANs sur les trois switches via le domaine VTP `CISCO` (Server sur le switch de distribution, Client sur les switches d'accès).

---

### 2️⃣ Configurer les ports d'accès en mode `access` avec le bon VLAN + `spanning-tree portfast` et `bpduguard`

![Configuration des ports access, trunk et port-security](06_switches_access_trunk_and_port_security_config.png)

Extrait de configuration : les ports `FastEthernet0/3` à `0/6` sont affectés respectivement aux VLANs 10, 20, 30 et 99 en mode `switchport mode access`, avec `switchport port-security mac-address sticky` activé. Les ports `Fa0/1-Fa0/2` sont en trunk vers le switch de distribution.

> ℹ️ Le `portfast` et le `bpduguard` ne sont pas visibles dans cette capture — à ajouter/vérifier séparément (voir section Points d'amélioration).

---

### 3️⃣ Configurer les deux liens SW-DIST/SW-ACCES1 en Port-Channel LACP actif (mode active), puis en trunk 802.1Q

![Trunks et configuration L3 du switch de distribution](04_multilayer_switch_trunk_and_l3_config.png)

Sur le switch de distribution, les interfaces `GigabitEthernet1/0/1-2` sont configurées en trunk (`switchport mode trunk`, `switchport nonegotiate`, VLAN natif 8, VLANs autorisés 8,10,20,30,99).

> ℹ️ Le regroupement en **EtherChannel LACP (`channel-group ... mode active`)** n'apparaît pas explicitement dans les captures actuelles de ce lab — ces deux liens sont configurés en trunks individuels. Voir section Points d'amélioration pour la commande à ajouter.

---

### 4️⃣ Configurer le lien SW-DIST/SW-ACCES2 en simple trunk 802.1Q, encapsulation dot1q

![Show interfaces trunk / VTP sur les 3 switches](03_switches_show_interfaces_trunk_vtp.png)

`SH INT TRUNK` confirme l'encapsulation **802.1q** sur les liens `Fa0/1-Fa0/2` (accès) et `Gi1/0/1-Gi1/0/2` (distribution), avec VLAN natif 8 et VLANs autorisés 8,10,20,30,99 cohérents sur toute la chaîne.

---

### 5️⃣ Fixer SW-DIST comme root bridge (priorité 4096) pour tous les VLANs et vérifier avec `show spanning-tree`

![Configuration trunk et L3 du switch de distribution](04_multilayer_switch_trunk_and_l3_config.png)

Le mode `spanning-tree mode pvst` est activé sur le switch de distribution.

> ℹ️ La commande de priorité root bridge (`spanning-tree vlan 1,10,20,30,99 priority 4096`) et la sortie de `show spanning-tree` ne sont pas présentes dans les captures fournies — à documenter séparément.

---

### 6️⃣ Créer les SVI (VLAN 10, 20, 99, 1) sur SW-DIST et activer le routage IP (`ip routing`)

![SVI et ip helper-address sur le switch de distribution](11_multilayer_switch_svi_and_ip_helper_config.png)

Les interfaces virtuelles `Vlan10`, `Vlan20`, `Vlan30` et `Vlan99` sont créées avec une adresse MAC dédiée, une adresse IP de passerelle, et un `ip helper-address 10.0.0.1` pour relayer les requêtes DHCP vers le routeur.

**Côté routeur**, les sous-interfaces en Router-on-a-Stick complètent le schéma de routage inter-VLAN :

![Sous-interfaces 802.1Q du routeur](10_router1_subinterfaces_dot1q_config.png)

`GigabitEthernet0/0.10`, `.20`, `.30` et `.99` avec `encapsulation dot1Q` et adresse IP par VLAN.

![Interfaces globales du routeur](07_router1_interfaces_global_config.png)

Interface `GigabitEthernet0/0` configurée en lien routé point-à-point (`10.0.0.1/30`) vers le switch de distribution.

---

### 7️⃣ Configurer le pool DHCP sur SW-DIST pour chaque VLAN, avec exclusions des adresses de gestion

![Pools DHCP et adresses exclues sur le routeur](08_router1_dhcp_pools_and_excluded_addresses.png)

Quatre pools DHCP (`DHCP_10`, `DHCP_20`, `DHCP_30`, `DHCP_99`) sont configurés avec :
- Exclusion des adresses réseau, passerelle et DNS de chaque plage,
- `default-router` pointant vers la SVI correspondante,
- `dns-server` et `domain-name LAB.NET`.

![Routes statiques du routeur](05_router1_static_routes_config.png)

Les routes statiques vers les réseaux VLAN (192.168.10.0, .20.0, .30.0, .99.0) passent par `10.0.0.2` (switch de distribution), assurant l'acheminement des retours DHCP relayés.

---

### 8️⃣ Vérifier la connectivité inter-VLAN et l'obtention d'adresses DHCP sur chaque PC

![Vérification IP DHCP sur les PC clients](09_clients_pc_dhcp_ip_configuration_verification.png)

Les 8 PC (PC0 à PC7) ont bien obtenu une adresse IPv4 automatiquement via DHCP, cohérente avec leur VLAN d'appartenance (ex : PC0 → 192.168.10.3, PC1 → 192.168.20.3, PC3 → 192.168.99.3, etc.).

![Tests ping inter-VLAN entre les PC](13_test_ping_inter-vlan_pc_clients.png)

Tests `ping` croisés entre VLANs différents (10↔20↔30↔99) : la connectivité inter-VLAN fonctionne, avec quelques pertes initiales dues à la résolution ARP (normal au premier échange).

---

## ✅ Résultats obtenus

- Les VLANs de service et le VLAN management sont créés, nommés et isolés au niveau 2.
- Les trunks 802.1Q filtrent correctement les VLANs autorisés (pruning effectif, VLAN natif dédié).
- Le routage inter-VLAN via les SVI fonctionne (pings inter-VLAN réussis).
- Le DHCP relayé (`ip helper-address`) attribue correctement les adresses aux postes clients selon leur VLAN.
- La sécurité des ports (port-security sticky) est activée sur les ports d'accès.

## 🔧 Points d'amélioration / éléments non couverts par les captures actuelles

- [ ] **EtherChannel LACP** : ajouter `channel-group 1 mode active` sur les deux liens SW-DIST ↔ SW-ACCES1 avant de les mettre en trunk.
- [ ] **Root bridge STP** : configurer `spanning-tree vlan 1,10,20,30,99 priority 4096` sur SW-DIST et fournir la sortie `show spanning-tree` pour vérification.
- [ ] **Portfast / BPDU Guard** : ajouter `spanning-tree portfast` et `spanning-tree bpduguard enable` sur les ports d'accès utilisateurs.
- [ ] Fournir une capture de `show etherchannel summary` et `show spanning-tree vlan <id>` une fois ces points implémentés.

## 📂 Index des captures

| # | Fichier | Tâche associée |
|---|---|---|
| 1 | `01_topologie_reseau_packet_tracer.png` | Schéma d'architecture |
| 2 | `02_switches_show_vlan_brief.png` | Tâche 1 — Création des VLANs |
| 3 | `03_switches_show_interfaces_trunk_vtp.png` | Tâche 4 — Trunk 802.1Q |
| 4 | `04_multilayer_switch_trunk_and_l3_config.png` | Tâches 3 & 5 — Trunk LACP / STP root |
| 5 | `05_router1_static_routes_config.png` | Tâche 7 — Routes vers pools DHCP |
| 6 | `06_switches_access_trunk_and_port_security_config.png` | Tâche 2 — Ports d'accès |
| 7 | `07_router1_interfaces_global_config.png` | Tâche 6 — Lien routé vers SW-DIST |
| 8 | `08_router1_dhcp_pools_and_excluded_addresses.png` | Tâche 7 — Pools DHCP |
| 9 | `09_clients_pc_dhcp_ip_configuration_verification.png` | Tâche 8 — Vérification DHCP |
| 10 | `10_router1_subinterfaces_dot1q_config.png` | Tâche 6 — SVI / sous-interfaces |
| 11 | `11_multilayer_switch_svi_and_ip_helper_config.png` | Tâche 6 — SVI + ip helper-address |
| 12 | `12_switches_vtp_status_and_vlan_brief.png` | Tâche 1 — Vérification VTP/VLAN |
| 13 | `13_test_ping_inter-vlan_pc_clients.png` | Tâche 8 — Ping inter-VLAN |
