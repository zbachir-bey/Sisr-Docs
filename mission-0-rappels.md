# Mission 0 — Rappels et contexte

## E.1.1 Rappels du premier semestre

### Architecture réseau

- 1 serveur sur le réseau `10.31.16.0/20`
- 1 routeur reliant les réseaux `172.31.16.0/16` et `10.31.16.0/20`
- 1 switch d'interconnexion entre le serveur et le routeur

---

### Calculs d'adressage — 10.31.16.0/20

**Nombre d'hôtes :**

$$2^{32-20} - 2 = 4094 \text{ machines}$$

**Calcul du broadcast :**

```
  10.31.16.0   →  00001010  00011111  00010000  00000000  (IP)
  Masque /20   →  11111111  11111111  11110000  00000000  (Masque)
               ──────────────────────────────────────────
  Broadcast    →  00001010  00011111  00011111  11111111
```

> **Broadcast : `10.31.31.255`**
> **Adresse routeur : `10.31.31.254`**

---

### Plan d'adressage IP

| Équipement | Interface | Adresse IP | Passerelle |
| :--- | :--- | :--- | :--- |
| Routeur | LAN | `10.31.31.254/20` | — |
| Routeur | WAN | `172.31.16.254/16` | `172.31.0.1` |
| Serveur | LAN | `10.31.16.1/20` | `10.31.31.254` |
| Routeur Prof | WAN | `172.31.0.1/16` | — |
| Pare-feu Beaupeyrat | — | `10.187.20.254/24` | — |

---

### Passerelles par défaut

| Machine | Passerelle |
| :--- | :--- |
| Serveur | `10.31.31.254` |
| Routeur | `172.31.0.1` |

---

## E.1.2 Schéma complet du réseau

![Schéma réseau](images/<img width="1029" height="728" alt="Capture d&#39;écran 2026-04-09 210014" src="https://github.com/user-attachments/assets/7e3d44be-7613-494b-b9bd-3c92f1c7271f" />
)
