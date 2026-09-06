# lmsctl

Pilotage en ligne de commande de **LM Studio en mode headless** (démon
`llmster`) sur Linux, via une unité systemd. Aucune dépendance à X11, Wayland
ou D-Bus : le serveur d'inférence démarre au boot, sans session graphique.

> Projet indépendant, sans lien avec LM Studio ni Element Labs. LM Studio est un
> logiciel propriétaire soumis à ses propres conditions d'utilisation ; ce script
> se contente d'invoquer son installateur et sa CLI officiels.

## Pourquoi

Faire tourner LM Studio sur un serveur d'inférence dédié suppose normalement une
session graphique et une AppImage lancée à la main. Le démon `llmster` supprime
ce besoin, mais son intégration système reste à écrire : unité systemd, mises à
jour sans coupure hasardeuse, restauration quand une version ne démarre pas.
C'est ce que fait `lmsctl`.

## Prérequis

- Linux avec systemd, Bash **4.4 ou supérieur**
- `curl`, `tar`, `runuser` (util-linux), `ss` (iproute2), `find`, `sort`
- `libatomic` (paquet `libatomic1` sur Debian/Ubuntu), exigé par `llmster`
- Un compte non privilégié dédié, propriétaire de `~/.lmstudio`
- Les droits root pour l'installation et la gestion du service

La CI exécute ShellCheck et la suite de tests simulés sur Ubuntu 22.04 et 24.04.
Elle ne remplace pas un essai de déploiement avec un véritable systemd et une
installation `llmster`. Les autres distributions systemd devraient fonctionner ;
les retours sont bienvenus.

## Installation

```bash
sudo install -m 0755 lmsctl.sh /usr/local/sbin/lmsctl
sudo tee /etc/default/lmstudio-headless >/dev/null <<'EOF'
LMS_USER=inference
SERVER_PORT=1234
BIND_ADDR=127.0.0.1
EOF
sudo lmsctl check     # valide la configuration sans rien installer
sudo lmsctl install
```

`install` télécharge et vérifie l'installateur officiel, l'exécute sous le compte
cible, écrit l'unité systemd, l'active et attend que l'API réponde. `update`
emploie le même installateur prévalidé directement, sans lancer au préalable
`lms daemon update`. Cela évite un second téléchargement et les différences de
`PATH` que le processus interne de la CLI peut rencontrer pour des outils système
comme `ldconfig`.

Pour connaître la dernière version disponible, `update` récupère toujours le
petit script `install.sh` officiel. Il compare ensuite la valeur `APP_VERSION`
aux répertoires `~/.lmstudio/llmster/<version>`. Si la version annoncée contient
déjà un exécutable `llmster`, la commande s'arrête avant le téléchargement de
l'archive, la sauvegarde et l'arrêt du service. Un répertoire incomplet ne compte
pas comme une installation valide et sera réinstallé.

## Commandes

| Commande | Effet |
| --- | --- |
| `install` | Installe `llmster`, écrit et active l'unité, démarre le service |
| `update` | Met à jour si nécessaire ; sinon ne télécharge pas l'archive et ne redémarre rien |
| `configure` | Applique le fichier de configuration, redémarre si le service tourne |
| `check` | Valide configuration et unité générée, sans rien installer |
| `start` / `stop` / `restart` | Cycle de vie du service |
| `status` | État passif : service, démon, serveur, modèles, LM Link et sonde HTTP |
| `health` | Sonde HTTP seule, code de sortie exploitable en supervision |
| `logs [N]` | N dernières lignes du journal systemd |
| `models` | Modèles en mémoire et sur disque |
| `uninstall [--purge]` | Retire l'unité ; `--purge` supprime aussi les sauvegardes |

`status` n'appelle aucune sous-commande `lms` tant que la sonde HTTP n'a pas
confirmé qu'un serveur répond. Il ne réveille donc pas implicitement `llmster`
lorsque celui-ci est arrêté.

## Portée de l'unité systemd

L'unité suit le modèle headless officiel de LM Studio : `Type=oneshot` avec
`RemainAfterExit=yes`. systemd orchestre les commandes de démarrage et d'arrêt,
mais ne reste pas attaché au processus `llmster`. Un état `active (exited)`
signifie donc que la séquence de démarrage a réussi, pas que le démon est encore
sain plusieurs heures plus tard.

Cette unité n'est pas un watchdog et ne redémarre pas automatiquement `llmster`
s'il meurt après son lancement. Utilisez `lmsctl health` avec votre outil de
supervision si cette garantie est nécessaire.

## Configuration

Tout se règle dans `/etc/default/lmstudio-headless`, qui doit appartenir à root
et n'être modifiable ni par le groupe ni par les autres — le script le refuse
sinon, puisqu'il l'interprète comme du shell dans un processus privilégié.

| Variable | Défaut | Rôle |
| --- | --- | --- |
| `LMS_USER` | `$SUDO_USER` | Compte propriétaire de l'installation |
| `SERVICE_NAME` | `lmstudio` | Nom de l'unité systemd |
| `SERVER_PORT` | `1234` | Port d'écoute de l'API |
| `BIND_ADDR` | `127.0.0.1` | Adresse d'écoute ; `0.0.0.0` pour le réseau local |
| `PRELOAD_MODEL` | *(vide)* | Modèle chargé au démarrage du service |
| `LOAD_ARGS` | *(vide)* | Arguments de `lms load`, ex. `--context-length 32768` |
| `HEALTH_TIMEOUT` | `180` | Attente maximale de l'API après un démarrage |
| `ENABLE_LM_LINK` | `false` | Réactive LM Link après chaque démarrage ; le compte `LMS_USER` doit d'abord y être connecté |
| `API_TOKEN_FILE` | *(vide)* | Fichier root:root 0600 dont le jeton est envoyé par la sonde HTTP ; il n'active pas l'authentification à lui seul |
| `FAILED_RETENTION` | `3` | Quarantaines d'échec conservées |

### LM Link sur un serveur headless

LM Link permet à LM Studio sur un poste client (par exemple un Mac) de découvrir
ce serveur et d'exécuter à distance le chargement et l'inférence. La liaison est
chiffrée de bout en bout et ne demande **ni** d'exposer l'API du serveur sur le
réseau, **ni** d'ouvrir `SERVER_PORT` dans le pare-feu. Gardez donc :

```bash
BIND_ADDR=127.0.0.1
ENABLE_LM_LINK=true
```

Le client continue de parler à son propre serveur LM Studio local, généralement
via `http://127.0.0.1:1234/v1`. LM Link transporte ensuite la requête vers le
modèle hébergé sur le serveur : l'adresse et le port du serveur ne sont pas
configurés dans l'outil client. Sur le Mac, activez LM Link avec le même compte,
vérifiez que le serveur apparaît, puis sélectionnez ses modèles distants ou
faites-en le périphérique préféré.

Après le premier `sudo lmsctl install`, connectez une fois le compte de service
à LM Link. Remplacez `inference` par la valeur de `LMS_USER` :

```bash
sudo -u inference -H /home/inference/.lmstudio/bin/lms login
# Suivre le parcours de connexion indiqué dans le terminal.

sudo -u inference -H /home/inference/.lmstudio/bin/lms link enable
sudo -u inference -H /home/inference/.lmstudio/bin/lms \
  link set-device-name "inference-01"
sudo -u inference -H /home/inference/.lmstudio/bin/lms link status
```

`ENABLE_LM_LINK=true` fait ensuite exécuter `lms link enable` après chaque
démarrage du serveur. Cette tentative est limitée à 20 secondes et ne bloque pas
l'API si aucun autre appareil n'est connecté. `lmsctl status` affiche l'état de
la liaison une fois l'API locale disponible.

### Exposition directe de l'API

Cette section ne concerne que les clients qui doivent joindre directement
`http://<serveur>:<SERVER_PORT>/v1`. Elle est indépendante de LM Link.

`BIND_ADDR=0.0.0.0` rend l'API joignable depuis le réseau. Activez alors
l'authentification dans LM Studio et placez le jeton dans `API_TOKEN_FILE` pour
que la sonde reste authentifiée. Ce fichier est lu par `lmsctl` uniquement : il
ne crée ni n'active un jeton côté LM Studio. Filtrez également le port, en
adaptant le réseau et le port à votre configuration :

```bash
sudo ufw allow from 192.168.1.0/24 to any port 1234 proto tcp
```

N'utilisez pas `BIND_ADDR=0.0.0.0` uniquement pour LM Link. Si l'API directe
doit être accessible au-delà d'un réseau de confiance, placez plutôt un proxy
authentifié et chiffré devant elle.

## Codes de sortie

| Code | Signification |
| --- | --- |
| `0` | Opération réussie et vérifiée |
| `1` | Échec ; une restauration ciblée a pu être tentée |
| `2` | Opération conservée, mais contrôle authentifié impossible (HTTP 401/403) |

Le code `2` est volontairement distinct : une sonde rejetée prouve que le serveur
répond. Un wrapper en `set -e` doit le traiter comme un avertissement.

## Ce que couvre la restauration

Avant toute mutation, `install` et `update` copient `~/.lmstudio/bin` dans
`/var/backups/<service>-llmster` et relèvent l'inventaire de
`~/.lmstudio/llmster/<version>`. Si l'installation, la génération de l'unité, le
démarrage ou la sonde échouent, une unique fonction `recover` restaure `bin/`,
écarte les versions de démon apparues pendant l'opération, remet l'unité et son
état `enabled` d'origine, puis redémarre le service s'il tournait auparavant.

Pendant une restauration, les artefacts fautifs de l'exécution courante sont
déplacés dans une nouvelle quarantaine horodatée sans écraser une quarantaine
précédente. Après une opération réussie, les quarantaines les plus anciennes
sont supprimées au-delà de `FAILED_RETENTION`. `uninstall --purge` supprime
également l'ensemble des sauvegardes et quarantaines.

**Limite assumée** : ce n'est pas un instantané complet de `~/.lmstudio`. L'état
interne du démon (`.internal`, caches, index de modèles) n'est pas versionné, et
l'installateur officiel ne sert que la dernière version — un retour à une version
arbitraire n'est donc pas possible. L'inventaire suit les noms des répertoires
`llmster/<version>` : une réinstallation qui modifierait en place un répertoire
portant exactement le même numéro de version ne pourrait pas restaurer son
contenu antérieur. La copie de `bin/` et l'unité systemd restent, elles,
restaurées.

## Développement

```bash
shellcheck -s bash lmsctl.sh tests/run-tests.sh
bash tests/run-tests.sh ./lmsctl.sh
```

La suite ne demande ni root, ni systemd, ni `llmster` : chaque cas source le
script dans un sous-shell et substitue les appels privilégiés. Les scénarios de
restauration et les parcours `install`, `update` et `configure` s'exécutent sur
une arborescence temporaire avec des fac-similés de `systemctl` et de la CLI
`lms`.

Toute modification touchant `recover`, la sonde de santé ou la génération de
l'unité doit venir avec son cas de test : ce sont les chemins qu'on ne peut pas
vérifier à la main sans provoquer de vraie panne.

## Licence

MIT — voir [LICENSE](LICENSE).
