# Mission 1 — Installation des systèmes

## Identifiants

> ⚠️ Les mots de passe ci-dessous sont des exemples utilisés en environnement de TP.

| Machine | Login | Mot de passe |
| :--- | :--- | :--- |
| Routeur | `root` | `drowssap` |
| Routeur | `groupe16` | `drowssap` |
| Serveur | `root` | `drowssap` |
| Serveur | `groupe16` | `drowssap` |

---

## Configuration du routeur

Avant toute chose, vérifier sur quel port est branché le câble réseau.

**Assigner une adresse IP au routeur :**

```
ip addr add 172.31.16.254/16 dev enp4s0
```

**Se connecter au compte root depuis l'utilisateur :**

```
su -
```

**Vérifier l'interface réseau :**

```
ip a
```

Le routeur a deux interfaces : `enp2s0` et `enp4s0`. L'interface `enp4s0` est reliée directement au réseau par le câble ethernet.

**Fichier `/etc/network/interfaces` du routeur :**

```
# The primary network interface
auto enp2s0
iface enp2s0 inet static
        address 10.31.31.254/20

auto enp4s0
iface enp4s0 inet static
        address 172.31.16.254/16
        gateway 172.31.0.1
        post-up iptables -t nat -A POSTROUTING -s 10.31.16.0/20 -o enp4s0 -j MASQUERADE
```

**Activer le routage** dans `/usr/lib/sysctl.d/50-default.conf` :

```
net.ipv4.ip_forward=1
```

**Définir le serveur DNS** dans `/etc/resolv.conf` :

```
nameserver 8.8.8.8
```

---

## Configuration du serveur

**Assigner une adresse IP au serveur :**

```
ip addr add 10.31.16.1/20 dev enp2s0
```

**Se connecter au compte root depuis l'utilisateur :**

```
su -
```

**Vérifier l'interface réseau :**

```
ip a
```

Le serveur a une seule interface : `eno1`.

![Vérification interface réseau du serveur](images/Capture%20d%27%C3%A9cran%202026-04-09%20222446.png)

**Fichier `/etc/network/interfaces` du serveur :**

```
# The primary network interface
auto eno1
iface eno1 inet static
        address 10.31.16.1/20
        gateway 10.31.31.254
```

**Définir le serveur DNS** dans `/etc/resolv.conf` :

```
nameserver 8.8.8.8
```

---

## Accès à distance depuis mon PC personnel

Pour accéder au réseau étudiant depuis ma machine, ajouter une route statique vers le routeur prof.

**Windows :**

```
route add -p 10.31.0.0 mask 255.255.0.0 10.187.20.10
route add -p 172.31.0.0 mask 255.255.0.0 10.187.20.10
```

**Linux :**

```
route add 10.31.0.0/16 gw 10.187.20.10
route add 172.31.0.0/16 gw 10.187.20.10
```

---

## Connexion par SSH

**Se connecter au serveur :**

```
ssh groupe16@10.31.16.1
```

![Connexion SSH au serveur](images/Capture%20d%27%C3%A9cran%202026-04-07%20164017.png)

**Se connecter au routeur :**

```
ssh groupe16@172.31.16.254
```
