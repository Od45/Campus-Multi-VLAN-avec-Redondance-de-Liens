# 🏢 Campus Multi-VLAN avec Redondance de Liens

**Niveau :** Intermédiaire
**Outils :** Cisco Packet Tracer
**Thèmes couverts :** VLAN · Trunking 802.1Q · STP Rapid-PVST+ · EtherChannel (LACP) · DHCP (relay/ip helper) · Routage inter-VLAN (SVI)

---

## 📋 Contexte

Une PME répartie sur deux étages doit segmenter son trafic par service (**Direction, Comptabilité, Ventes, Invités/Management**) tout en garantissant une continuité de service en cas de panne d'un lien entre switches. Ce lab met en œuvre une architecture **collapsed-core** avec un switch de distribution niveau 3 (SVI + routage inter-VLAN), deux switches d'accès, et un routeur en amont assurant le relais DHCP et le routage vers l'extérieur via des sous-interfaces 802.1Q.

## 🗺️ Topologie

![Topologie réseau](01_topologie_reseau_packet_tracer.png)

```
                     ┌────────────┐
                     │  Router1   │  (Gi0/0 en trunk .10/.20/.30/.99)
                     └─────┬──────┘
                            │ Trunk 802.1Q
                     ┌─────┴──────────┐
                     │ Multilayer SW0 │  (L3 - SVI + DHCP relay)
                     │   3650-24PS    │
                     └───┬────────┬───┘
             Trunk 802.1Q│        │Trunk 802.1Q
                ┌────────┘        └────────┐
           ┌────┴────┐              ┌──────┴───┐
           │ Switch0 │◄────────────►│ Switch1  │  (lien redondant entre accès)
           │2960-24TT│  Trunk 802.1Q│2960-24TT │
           └────┬────┘              └────┬─────┘
          PC0-PC3 (VLAN 10/20/30/99) PC4-PC7 (VLAN 99/30/20/10)
```

## 🔢 Plan d'adressage

| VLAN | Nom          | Réseau            | Passerelle (SVI) | Rôle                  |
|------|--------------|-------------------|-------------------|-----------------------|
| 10   | DIRECTION    | 192.168.10.0/27   | 192.168.10.2      | Poste Direction       |
| 20   | COMPTA       | 192.168.20.0/27   | 192.168.20.2      | Poste Comptabilité    |
| 30   | VENTES       | 192.168.30.0/27   | 192.168.30.2      | Poste Ventes          |
| 99   | MANAGEMENT   | 192.168.99.0/28   | 192.168.99.2      | Gestion / Invités     |
| —    | Backbone L3  | 10.0.0.0/30       | —                 | Liaison Router1 ↔ SW-DIST |

Adressage attribué **dynamiquement par DHCP** aux postes clients (voir capture PC).

---

## 🧭 Cahier des tâches et implémentation

### 1️⃣ Création des VLANs

VLANs créés et nommés sur les 3 switches (`SH VLAN` / `SH VLAN BRIEF`) :

| Fichier | Contenu |
|---|---|
| `02_switches_show_vlan_brief.png` | Vérification des VLANs 10/20/30/99 sur Switch0, Multilayer Switch0 et Switch1, avec répartition des ports par VLAN |

### 2️⃣ Trunking 802.1Q & VTP

Domaine VTP `CISCO` (mode Server sur SW-DIST, Client sur les switches d'accès), trunks configurés en `dot1q`, `native vlan 8`, VLANs autorisés filtrés (pruning) :

| Fichier | Contenu |
|---|---|
| `03_switches_show_interfaces_trunk_vtp.png` | `SH INT TRUNK` + `SH VTP STATUS` : encapsulation 802.1Q, native VLAN 8, VLANs autorisés 8,10,20,30,99 |
| `12_switches_vtp_status_and_vlan_brief.png` | Vérification croisée VTP + VLAN brief sur les 3 équipements |

### 3️⃣ Configuration des ports (access, trunk, port-security)

| Fichier | Contenu |
|---|---|
| `06_switches_access_trunk_and_port_security_config.png` | Ports d'accès configurés par VLAN (Fa0/3→VLAN10, Fa0/4→VLAN20, Fa0/5→VLAN30, Fa0/6→VLAN99) avec `switchport port-security mac-address sticky` ; trunks Fa0/1-Fa0/2 en mode trunk natif VLAN 8 |

### 4️⃣ Switch de distribution (L3) — Trunk & routage

| Fichier | Contenu |
|---|---|
| `04_multilayer_switch_trunk_and_l3_config.png` | Interfaces Gi1/0/1-2 et Gi1/0/4-8 en trunk `nonegotiate`, VLAN natif 8 ; interface routée Gi1/0/3 en `no switchport` (liaison L3 point-à-point vers Router1, 10.0.0.2/30) ; `spanning-tree mode pvst` |

### 5️⃣ Interfaces virtuelles (SVI) & relais DHCP

| Fichier | Contenu |
|---|---|
| `11_multilayer_switch_svi_and_ip_helper_config.png` | SVI VLAN10/20/30/99 avec `mac-address` dédiée, IP de passerelle, et `ip helper-address 10.0.0.1` pour relayer les requêtes DHCP vers le routeur ; `ip classless` |

### 6️⃣ Routeur — Sous-interfaces 802.1Q (Router-on-a-Stick / uplink L3)

| Fichier | Contenu |
|---|---|
| `10_router1_subinterfaces_dot1q_config.png` | Sous-interfaces Gi0/0.10/.20/.30/.99 avec `encapsulation dot1Q` + IP par VLAN (rôle de passerelle secondaire / interco) |
| `07_router1_interfaces_global_config.png` | Interface Gi0/0 configurée en `10.0.0.1/30` (lien routé vers SW-DIST) |
| `05_router1_static_routes_config.png` | Routes statiques vers les réseaux VLAN 10/20/30/99 via `10.0.0.2` (next-hop = SW-DIST) |

### 7️⃣ Service DHCP centralisé

| Fichier | Contenu |
|---|---|
| `08_router1_dhcp_pools_and_excluded_addresses.png` | Pools DHCP `DHCP_10/20/30/99` avec exclusions des adresses réseau/gateway/broadcast, `default-router`, `dns-server`, `domain-name LAB.NET` |

### 8️⃣ Vérification de la connectivité inter-VLAN et DHCP

| Fichier | Contenu |
|---|---|
| `09_clients_pc_dhcp_ip_configuration_verification.png` | PC0 à PC7 : adresses IPv4 obtenues automatiquement par DHCP, cohérentes avec leur VLAN d'appartenance |
| `13_test_ping_inter-vlan_pc_clients.png` | Tests `ping` croisés entre VLANs (10↔20↔30↔99) : succès avec pertes ponctuelles liées à la résolution ARP initiale |

---

## 💻 Commandes CLI reconstituées

### Switches d'accès (Switch0 / Switch1)

```
enable
configure terminal
hostname Switch

vlan 10
 name DIRECTION
vlan 20
 name COMPTA
vlan 30
 name VENTES
vlan 99
 name MANAGEMENT
exit

spanning-tree mode pvst
spanning-tree extend system-id

! Ports d'accès
interface FastEthernet0/3
 switchport mode access
 switchport access vlan 10
 switchport port-security mac-address sticky

interface FastEthernet0/4
 switchport mode access
 switchport access vlan 20
 switchport port-security mac-address sticky

interface FastEthernet0/5
 switchport mode access
 switchport access vlan 30
 switchport port-security mac-address sticky

interface FastEthernet0/6
 switchport mode access
 switchport access vlan 99
 switchport port-security mac-address sticky

! Liens trunk vers le switch de distribution
interface range FastEthernet0/1-2
 switchport trunk native vlan 8
 switchport trunk allowed vlan 8,10,20,30,99
 switchport mode trunk

end
write memory
```

### Switch de distribution (Multilayer Switch0 - L3)

```
enable
configure terminal
hostname MultilayerSwitch0

ip routing
spanning-tree mode pvst

vlan 10
 name DIRECTION
vlan 20
 name COMPTA
vlan 30
 name VENTES
vlan 99
 name MANAGEMENT
exit

! Interfaces vers les switches d'accès (trunk)
interface range GigabitEthernet1/0/1-2
 switchport trunk native vlan 8
 switchport trunk allowed vlan 8,10,20,30,99
 switchport mode trunk
 switchport nonegotiate

interface range GigabitEthernet1/0/4-8
 switchport mode trunk
 switchport nonegotiate

! Interface routée vers Router1
interface GigabitEthernet1/0/3
 no switchport
 ip address 10.0.0.2 255.255.255.252
 duplex auto
 speed auto

! SVI (interfaces virtuelles routées)
interface Vlan10
 mac-address 0060.5c1e.5a01
 ip address 192.168.10.2 255.255.255.224
 ip helper-address 10.0.0.1

interface Vlan20
 mac-address 0060.5c1e.5a02
 ip address 192.168.20.2 255.255.255.224
 ip helper-address 10.0.0.1

interface Vlan30
 mac-address 0060.5c1e.5a03
 ip address 192.168.30.2 255.255.255.224
 ip helper-address 10.0.0.1

interface Vlan99
 mac-address 0060.5c1e.5a04
 ip address 192.168.99.2 255.255.255.240
 ip helper-address 10.0.0.1

ip classless
end
write memory
```

### Router1

```
enable
configure terminal
hostname Router

! Interface routée vers SW-DIST
interface GigabitEthernet0/0
 ip address 10.0.0.1 255.255.255.252
 duplex auto
 speed auto

! Sous-interfaces 802.1Q (si Router-on-a-Stick complémentaire)
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.224

interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.224

interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.224

interface GigabitEthernet0/0.99
 encapsulation dot1Q 99
 ip address 192.168.99.1 255.255.255.240

! Routes statiques vers les réseaux VLAN via SW-DIST
ip classless
ip route 192.168.10.0 255.255.255.224 10.0.0.2
ip route 192.168.20.0 255.255.255.224 10.0.0.2
ip route 192.168.30.0 255.255.255.224 10.0.0.2
ip route 192.168.99.0 255.255.255.240 10.0.0.2

! Configuration DHCP centralisée
ip dhcp excluded-address 192.168.10.1 192.168.10.2
ip dhcp excluded-address 192.168.10.9
ip dhcp excluded-address 192.168.20.1 192.168.20.2
ip dhcp excluded-address 192.168.20.9
ip dhcp excluded-address 192.168.30.1 192.168.30.2
ip dhcp excluded-address 192.168.30.9
ip dhcp excluded-address 192.168.99.1 192.168.99.2
ip dhcp excluded-address 192.168.99.9

ip dhcp pool DHCP_10
 network 192.168.10.0 255.255.255.224
 default-router 192.168.10.2
 dns-server 192.168.10.9
 domain-name LAB.NET

ip dhcp pool DHCP_20
 network 192.168.20.0 255.255.255.224
 default-router 192.168.20.2
 dns-server 192.168.20.9
 domain-name LAB.NET

ip dhcp pool DHCP_30
 network 192.168.30.0 255.255.255.224
 default-router 192.168.30.2
 dns-server 192.168.30.9
 domain-name LAB.NET

ip dhcp pool DHCP_99
 network 192.168.99.0 255.255.255.240
 default-router 192.168.99.2
 dns-server 192.168.99.9
 domain-name LAB.NET

end
write memory
```

### Commandes de vérification

```
show vlan brief
show interfaces trunk
show vtp status
show spanning-tree
show etherchannel summary
show ip route
show ip interface brief
show standby
show port-security interface <int>
show running-config
```

---

## ✅ Résultats obtenus

- Les **3 VLANs de service + VLAN management** sont opérationnels et isolés au niveau 2.
- Les **trunks 802.1Q** filtrent bien les VLANs autorisés (pruning effectif).
- Le **routage inter-VLAN** via les SVI du switch de distribution fonctionne (pings inter-VLAN réussis).
- Le **DHCP relayé** (`ip helper-address`) attribue correctement les adresses aux postes clients selon leur VLAN.
- La **sécurité des ports** (port-security sticky) est activée sur les ports d'accès.

## 🔧 Points d'amélioration possibles

- [ ] Ajouter l'**EtherChannel LACP** (`channel-group mode active`) sur les liens SW-DIST ↔ SW-ACCES1, non visible dans les captures actuelles.
- [ ] Configurer explicitement **Rapid-PVST+** et fixer la **priorité root bridge (4096)** sur le switch de distribution pour tous les VLANs.
- [ ] Activer **`spanning-tree portfast` + `bpduguard`** sur les ports utilisateurs.
- [ ] Sécuriser le VLAN natif (déjà en VLAN 8, dédié — bonne pratique confirmée).
- [ ] Documenter la table de routage complète (`show ip route`) sur SW-DIST et Router1.

## 📂 Structure des captures

```
01_topologie_reseau_packet_tracer.png
02_switches_show_vlan_brief.png
03_switches_show_interfaces_trunk_vtp.png
04_multilayer_switch_trunk_and_l3_config.png
05_router1_static_routes_config.png
06_switches_access_trunk_and_port_security_config.png
07_router1_interfaces_global_config.png
08_router1_dhcp_pools_and_excluded_addresses.png
09_clients_pc_dhcp_ip_configuration_verification.png
10_router1_subinterfaces_dot1q_config.png
11_multilayer_switch_svi_and_ip_helper_config.png
12_switches_vtp_status_and_vlan_brief.png
13_test_ping_inter-vlan_pc_clients.png
```
