# Configuration SSH

## Création des paires de clés

Depuis le terminal en administrateur :

```
ssh-keygen -t rsa -b 4096
```

Cela génère deux fichiers :

- `C:\Users\XXXXX\.ssh\id_rsa` — clé privée
- `C:\Users\XXXXX\.ssh\id_rsa.pub` — clé publique

---

## Enregistrement des clés publiques sur le serveur et le routeur

Vérifier l'existence du fichier `authorized_keys` dans le répertoire `.ssh`. S'il n'existe pas, le créer :

```
nano authorized_keys
```

![Création du fichier authorized_keys](images/Capture%20d%27%C3%A9cran%202026-04-08%20205032.png)

Copier le contenu de `id_rsa.pub` dans `/root/.ssh/authorized_keys` sur le serveur et le routeur.

Clés publiques enregistrées :

| Utilisateur | Machine |
| :--- | :--- |
| Zakaria | `pedago\zbachirbey@ELOI04` |
| Abdine | `abdine@Abdine` |
| backup1 | `root@backup1` |
| backup | `backup1@backup` |
