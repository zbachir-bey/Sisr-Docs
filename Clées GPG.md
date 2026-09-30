# GPG

On commence par installer le service gpg :

```bash
apt update
apt install gpg
```

Puis créez **`/home/.gnupg/gpg.conf`** :

```bash
cd /home
mkdir .gnupg
cd .gnupg
nano gpg.conf
```

Une clé GPG porte jusqu'à quatre capacités :

| Capacité | Rôle |
|---|---|
| **C** - Certifier | Attester que telle clé appartient à telle personne (dont ses propres sous-clés) |
| **S** - Signer | Prouver l'origine et l'intégrité d'un document |
| **E** - Chiffrer | Rendre un document lisible par un seul destinataire (Encrypt) |
| **A** - Authentifier | Prouver son identité à un service - c'est celle qui servira pour SSH |

## Étape 0 - Reconnaissance

Avant de créer quoi que ce soit, on regarde ce que sait faire l'outil.

```bash
gpg --version
gpg -k
gpg -K
```

`-k` liste les clés publiques du trousseau, `-K` les clés privées. Au départ, les deux sont vides.

## Étape 1 - La clé maître

L'idée : la clé maître ne doit servir **qu'à certifier**. Elle reste "hors ligne" une fois les sous-clés créées, c'est elle qui représente notre identité.

```bash
gpg --full-generate-key --expert
```

On choisit :

- type de clé : RSA (certification uniquement)
- taille : 4096
- validité : 30 jours (clé jetable pour ce TP)
- identité : Zakaria / zak@sisr2.org

Vérification :

```bash
gpg -k
```

On doit voir la clé avec la seule capacité `[C]`.

## Étape 2 - Les trois sous-clés

On ajoute à la clé maître les capacités qui manquent (S, E, A), sans y toucher elle-même.

```bash
gpg --expert --edit-key "Prenom Nom"
gpg> addkey
```

On répète l'opération trois fois (signature, chiffrement, authentification), toujours en RSA 4096 / 30 jours. Chaque `addkey` ouvre le même menu de choix que pour la clé maître.

**Important :** il faut taper `save` à la fin, sinon rien n'est enregistré.

```bash
gpg> save
```

Vérification :

```bash
gpg -k
```

On doit voir une ligne `sub` par sous-clé, avec `[E]`, `[S]` et `[A]`.

**Check 1** : faire valider le trousseau par le prof.

## Étape 3 - Le certificat de révocation

C'est l'assurance-vie de la clé : si elle est compromise ou si on perd la passe-phrase, ce certificat permet d'annoncer qu'elle ne doit plus être utilisée.

```bash
gpg --gen-revoke --output revoc.asc KEYID
```

Ce certificat doit être stocké **séparément** de la clé privée (un vol donnerait à la fois le moyen d'usurper l'identité et celui de la détruire), mais il doit rester accessible même si la passe-phrase est perdue.

## Étape 4 - Export / import / clé maître qui s'absente

Un trousseau, ce n'est que des fichiers qu'on peut déplacer.

```bash
gpg -a --export KEYID > public.asc
gpg -a --export-secret-key KEYID > secret.asc
gpg -a --export-secret-subkeys KEYID > sub.asc
```

On supprime ensuite tout de la machine, dans l'ordre imposé (privé avant public) :

```bash
gpg --delete-secret-keys KEYID
gpg --delete-keys KEYID
gpgconf --kill gpg-agent
```

Puis on réimporte uniquement les sous-clés :

```bash
gpg --import sub.asc
gpg -K
```

La clé maître apparaît marquée `sec#` : le `#` signifie que la partie privée n'est pas présente. On peut toujours signer/chiffrer/déchiffrer/authentifier avec les sous-clés, mais plus certifier ni créer de nouvelles sous-clés.

**Check 2** : montrer le trousseau amputé de sa clé maître.

## Étape 5 - Échanger les clés

On transmet sa clé publique au binôme voisin (mail, clé USB...) et on importe la sienne :

```
Créér le fichier .asc avec sa clés publique puis faites :
gpg --import Abdine.asc
gpg -k
```

Puis on vérifie l'empreinte **de vive voix**, caractère par caractère :

```bash
gpg -k --with-fingerprint
```

Une empreinte reçue par mail ne prouve rien (le mail peut être intercepté ou usurpé) : seule une vérification par un canal différent de celui de l'envoi permet de détecter une clé substituée. Sa propre identité apparaît en `[ultime]` car on lui fait confiance par construction ; celle du correspondant reste non certifiée tant qu'on n'a pas validé son empreinte.

## Étape 6 - Chiffrer

```bash
gpg -r Abdine@sisr2.org --encrypt message.txt
```

On vérifie que le résultat (`Abdine.txt.gpg`) est illisible, puis on l'échange et on déchiffre celui reçu :

```bash
gpg --output Abdine.txt --decrypt Abdine.txt.gpg
```

Un fichier chiffré uniquement pour le correspondant n'est pas déchiffrable par soi-même, sauf à s'ajouter comme second destinataire (`-r` supplémentaire).

## Étape 7 - Signer

```bash
gpg --output doc.sig --sign doc.txt        # binaire, compressé
gpg --clearsign doc.txt                     # lisible, signature autour
gpg --detach-sign doc.txt                   # signature séparée
```

On transmet le document et sa signature détachée, puis on vérifie :

```bash
gpg --verify doc.txt.sig doc.txt
```

Un dépôt de paquets (.iso + .sig) utilise la signature détachée : le fichier original n'est pas modifié et reste utilisable tel quel, seule la signature est distribuée à côté.

## Étape 8 - Casser le sceau

On modifie un caractère du message dans un fichier `--clearsign` reçu, sans toucher au bloc de signature, puis on revérifie :

```bash
gpg --verify doc.txt.asc
```

La vérification échoue : une signature valide prouve l'intégrité et l'origine du document, pas la fiabilité de son auteur. Un avertissement du type "impossible de garantir que la clé appartient à cette personne" peut apparaître même avec une signature valide : la signature prouve que le document a été produit avec cette clé, pas que la clé appartient bien à qui on croit.

## Utiliser la sous-clé A pour SSH

gpg-agent sait parler le protocole de ssh-agent : SSH lui demande une signature, l'agent la produit avec la sous-clé `[A]`, et le serveur ne voit qu'une clé publique ordinaire dans `authorized_keys`.

### Côté client

On active le support SSH de l'agent :

```bash
echo enable-ssh-support >> ~/.gnupg/gpg-agent.conf
```

On récupère le keygrip de la sous-clé `[A]` (**pas son empreinte**) et on l'inscrit dans `sshcontrol` :

```bash
gpg -K --with-keygrip
echo <KEYGRIP_DE_LA_SOUS_CLE_A> >> ~/.gnupg/sshcontrol
```

On redirige SSH vers gpg-agent, dans `~/.bash_profile` :

```bash
unset SSH_AGENT_PID
if [ "${gnupg_SSH_AUTH_SOCK_by:-0}" -ne $$ ]; then
    export SSH_AUTH_SOCK="$(gpgconf --list-dirs agent-ssh-socket)"
fi
export GPG_TTY=$(tty)
gpg-connect-agent updatestartuptty /bye >/dev/null
```

On relance l'agent et on vérifie :

```bash
gpgconf --kill gpg-agent
gpg-connect-agent "getinfo ssh_socket_name" /bye
ssh-add -L
```

La sortie doit contenir une ligne `ssh-rsa AAAA…` ou `ssh-ed25519 AAAA…`. Si elle est vide, la suite ne fonctionnera pas.

**À partir de GnuPG 2.4 :** le fichier `sshcontrol` est déprécié au profit de l'attribut `Use-for-ssh` inscrit dans le fichier de la clé elle-même. Il reste lu pour compatibilité, donc la méthode ci-dessus fonctionne toujours — vérifier la version avec `gpg --version`.

### Côté serveur

On exporte la clé publique au format OpenSSH — le `!` désigne précisément la sous-clé `[A]` et non la clé maître :

```bash
gpg --export-ssh-key 0xEMPREINTE_SOUS_CLE_A! > id_gpg.pub
ssh-copy-id -f -i id_gpg.pub user@serveur
```

À la main, sur le serveur :

```bash
cat id_gpg.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Problème rencontré

Après avoir suivi la procédure, `ssh-add -L` renvoyait `communication with agent failed` puis `No such file or directory`. En tentant de lancer l'agent manuellement (`gpg-agent --daemon --verbose`), le vrai message d'erreur est apparu :

```
gpg-agent: /root/.gnupg/gpg-agent.conf:1: argument not expected
```

Le fichier `gpg-agent.conf` contenait une ligne corrompue (`enable-ssh-support -` au lieu de `enable-ssh-support`), probablement suite à un copier-coller incomplet. L'agent refusait de démarrer à cause de cette ligne invalide, ce qui rendait tout le socket SSH inutilisable en cascade.

**Résolution :**

```bash
echo "enable-ssh-support" > /root/.gnupg/gpg-agent.conf
gpgconf --kill gpg-agent
rm -f /run/user/0/gnupg/S.gpg-agent*
gpg-agent --daemon
source ~/.bash_profile
ssh-add -L
```

**Leçon retenue :** `gpgconf --kill gpg-agent` ne redémarre pas l'agent automatiquement en cas d'erreur de configuration — il faut le lancer explicitement pour voir le vrai message d'erreur, au lieu de se fier au code de retour générique renvoyé par `gpg-connect-agent`.

## Bonus - Révoquer

```bash
gpg --import revoc.asc
gpg -k
```

La clé apparaît marquée comme révoquée. On peut toujours déchiffrer les anciens messages reçus, mais plus en chiffrer de nouveaux pour cette clé. Les correspondants ne sont informés que si on republie la clé révoquée de leur côté.
