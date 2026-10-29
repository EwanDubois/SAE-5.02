# SAE-5.02
# Supervision déclarative des PMZ — SAÉ 5.02

Déploiement automatisé par Ansible d'une application web de suivi de l'état des PMZ
(points de mutualisation de zone).

L'application est volontairement minimale. L'essentiel du travail porte sur
l'automatisation : déploiement depuis un inventaire, secrets dans Vault, accès en
HTTPS, et sauvegarde quotidienne vers une seconde VM.

**Aucune donnée réelle d'Orange n'est utilisée.** Le référentiel (zones, PMZ, équipes
de dépannage) est entièrement fictif et décrit dans l'inventaire Ansible.

---

## 1. Environnement

### Machines virtuelles

| VM | Rôle | Adresse | Ressources |
| --- | --- | --- | --- |
| VM-app | application et conteneurs | 192.168.119.118 | 2 vCPU, 4 Go RAM, 30 Go |
| VM-backup | dépôt des sauvegardes | 192.168.119.218 | 1 vCPU, 1 Go RAM, 10 Go |

- Distribution : **Debian 12 (bookworm)**, installation minimale sans environnement graphique.
- Hyperviseur : VMware, les deux VM en mode pont sur le réseau 192.168.119.0/24.
- Nœud de contrôle Ansible : mon poste, hors VM.
- Les VM sont créées vierges à l'avance (Debian + SSH). Tout le reste est fait par
  `deploy.yml`.

### Images Docker

Les versions sont figées, aucune image en `latest`.

| Service | Image | Rôle |
| --- | --- | --- |
| `front` | `nginx:1.27-alpine` | reverse proxy, terminaison TLS, sert l'interface |
| `api` | `python:3.12-slim` | API REST, règles métier, envoi des demandes |
| `db` | `postgres:16.4-alpine` | référentiel et données métier |
| `mail` | `axllent/mailpit:v1.21` | serveur SMTP de test et boîte de réception |

Réseau Docker dédié : `172.20.0.0/24`. Seul le port 443 est publié sur VM-app.

---

## 2. État de l'application

**Le code applicatif n'existe pas encore, il est à développer dans le cadre du projet.**
Je préfère l'écrire clairement plutôt que de laisser croire à un existant.

Ce qui est fait à ce jour :

- dépôt initialisé, arborescence `inventory/` `roles/` `playbooks/` en place
- six rôles créés avec `ansible-galaxy init`
- inventaire d'exemple avec deux zones fictives
- syntaxe des playbooks vérifiée (`ansible-playbook --syntax-check`)

Ce qui reste à écrire, et qui est inclus dans l'estimation de 20 h :

- l'API FastAPI : trois endpoints (liste des PMZ, changement d'état, création de demande)
- le schéma PostgreSQL et les migrations
- l'interface cartographique

L'application reste minimale pour préserver le temps consacré à l'automatisation.
Pas d'authentification, pas d'alerte temps réel, pas d'indicateurs.

### Hébergement de l'interface cartographique

L'interface est une page statique (HTML + **Leaflet**) servie par le conteneur `front`.
Elle est déposée dans un volume par le rôle `app` et n'a pas de conteneur à elle.
Elle interroge l'API en `/api/`, que nginx relaie en interne.

**Limite connue** : Leaflet va chercher ses tuiles sur un serveur externe, ce qui suppose
un accès Internet depuis le navigateur de démonstration. Si l'accès est coupé, la carte
s'affiche vide. Deux pistes, par ordre de coût :

1. embarquer un fond de carte statique (image géoréférencée) dans le volume, ce qui
   suffit pour une zone fictive de quelques kilomètres carrés ;
2. ajouter un cinquième conteneur servant des tuiles locales, à ne faire que si le
   budget horaire le permet.

Je retiens la piste 1 par défaut.

---

## 3. Rôle `seed` : idempotence et séparation des données

C'est le point le plus délicat du projet. Une seconde exécution de `deploy.yml` ne doit
créer aucun doublon ni écraser ce qui a été saisi dans l'application.

### Deux catégories de données

**Référentiel, géré par Ansible.** Décrit dans les variables d'inventaire, recréable à
volonté :

- `zone` — code, libellé, commune fictive
- `pmz` — identifiant, coordonnées, capacité, locaux raccordables, zone de rattachement
- `equipe` — nom, adresse de contact collective, zone couverte

**Données métier, jamais touchées par Ansible.** Produites par les utilisateurs dans
l'application :

- `etat_pmz` — état courant de chaque PMZ et sa date de changement
- `ticket` — demandes d'intervention
- `historique` — transitions d'état
- `mail_archive` — mails réellement envoyés

### Comportement attendu

Le rôle `seed` fait un **upsert sur l'identifiant naturel** de chaque objet du
référentiel : il crée si absent, met à jour si présent, et ne supprime jamais.

```sql
INSERT INTO pmz (identifiant, libelle, latitude, longitude, capacite, locaux, zone_code)
VALUES (...)
ON CONFLICT (identifiant) DO UPDATE SET
  libelle = EXCLUDED.libelle,
  latitude = EXCLUDED.latitude,
  longitude = EXCLUDED.longitude,
  capacite = EXCLUDED.capacite,
  locaux = EXCLUDED.locaux,
  zone_code = EXCLUDED.zone_code;
```

Les tables `etat_pmz`, `ticket`, `historique` et `mail_archive` ne sont référencées dans
aucune tâche du rôle. Un PMZ déclaré en panne reste en panne après un redéploiement.

**Un piège que j'ai corrigé** : dans ma première version, l'état courant était une
colonne de la table `pmz`, donc l'upsert l'aurait écrasé à chaque passage. Je l'ai sorti
dans une table `etat_pmz` séparée. C'est ce qui rend la frontière nette entre les deux
catégories.

Une zone retirée de l'inventaire n'est pas supprimée en base, elle est marquée inactive.
Supprimer effacerait des tickets rattachés. C'est un choix assumé, au prix d'un
nettoyage manuel si le référentiel diverge beaucoup.

---

## 4. Accès à Mailpit pendant la démonstration

Mailpit écoute sur deux ports internes au réseau Docker : `1025` en SMTP pour l'API, et
`8025` pour son interface web. **Aucun des deux n'est publié sur VM-app**, conformément
à la règle « seul le 443 est ouvert ».

Pour consulter les messages pendant la démonstration, j'ouvre un tunnel SSH depuis mon
poste :

```bash
ssh -L 8025:127.0.0.1:8025 ewan@192.168.119.118
```

Puis `http://localhost:8025` dans le navigateur. Rien n'est exposé sur le réseau, la
restriction d'accès tient, et le tunnel se ferme avec la session SSH.

Côté VM, le port `8025` est republié sur la seule boucle locale (`127.0.0.1:8025`) et
non sur l'interface réseau, ce qui est ce qui rend le tunnel possible sans ouvrir quoi
que ce soit vers l'extérieur.

---

## 5. Sauvegarde et restauration

### Mécanisme

`backup.yml` lance un `pg_dump` dans le conteneur `db`, écrit l'archive horodatée sur
VM-app, puis la transfère vers VM-backup par `rsync` sur SSH avec une clé dédiée. Les
archives de plus de 14 jours sont supprimées sur VM-backup.

`restore.yml` prend une archive en paramètre, arrête l'API, restaure la base, puis
redémarre.

### Scénario de recette

C'est ce que je rejouerai en démonstration :

1. **Données connues** — déclarer une panne sur le PMZ `PMZ-A-003` et créer la demande
   associée. Noter la référence du ticket.
2. **Sauvegarde** — lancer `backup.yml`. Vérifier que l'archive est bien arrivée sur
   VM-backup et noter sa taille.
3. **Modification destructrice** — remettre le PMZ en service et supprimer le ticket
   directement en base. L'application ne montre plus rien.
4. **Restauration** — lancer `restore.yml` avec l'archive de l'étape 2.
5. **Contrôle** — le PMZ est de nouveau en panne, le ticket est revenu avec sa référence
   d'origine, et l'historique contient bien les deux transitions initiales.

### Vérifications automatisées

- le dump n'est pas vide (taille minimale) et `pg_restore --list` le relit sans erreur
- le nombre de lignes des tables métier est comparé avant et après restauration
- la date de l'archive la plus récente sur VM-backup est contrôlée

**Limite assumée** : les deux VM tournent sur le même poste physique. Une panne de son
disque emporte la sauvegarde avec l'original. La séparation démontre le mécanisme, pas
la résilience. Piste d'amélioration : une cible de sauvegarde hors du poste, un partage
réseau ou un stockage objet, hors périmètre du budget horaire.

---

## 6. Preuves d'idempotence

Les sorties sont conservées dans `docs/preuves/`, une par scénario, avec la date.

**Seconde exécution de `deploy.yml`** — la récapitulation doit afficher `changed=0` sur
les deux hôtes. Toute tâche qui reste en `changed` à la seconde passe est un défaut à
corriger, pas un résultat acceptable.

**Ajout d'une zone par les seules variables** — la procédure, reproductible :

1. sauvegarder l'état : nombre de PMZ en base et capture de la carte
2. ajouter une zone et ses PMZ dans `inventory/group_vars/all/referentiel.yml`
3. relancer `ansible-playbook playbooks/deploy.yml`
4. recharger la carte : les nouveaux PMZ apparaissent

Aucune modification du code applicatif, aucun `psql` manuel, aucune action dans
l'interface. Le `git diff` entre les deux états ne doit porter que sur le fichier de
variables, et c'est cette capture-là qui fait la preuve.

---

## 7. Arborescence

```
.
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       └── all/
│           ├── referentiel.yml    # zones, PMZ, équipes
│           └── vault.yml          # chiffré
├── roles/
│   ├── common/          # comptes, clés SSH, pare-feu
│   ├── docker/          # Docker Engine et Compose
│   ├── app/             # volumes, Compose généré en Jinja2, secrets
│   ├── seed/            # schéma et peuplement du référentiel
│   ├── reverse_proxy/   # nginx, TLS
│   └── backup/          # SSH inter-VM, dump quotidien, rétention
├── playbooks/
│   ├── deploy.yml
│   ├── backup.yml
│   ├── restore.yml
│   └── teardown.yml
├── app/                 # code applicatif (à développer)
└── docs/
    └── preuves/
```

---

## 8. Déploiement

```bash
# première fois
ansible-galaxy install -r requirements.yml
ansible-playbook playbooks/deploy.yml --ask-vault-pass

# vérifier l'idempotence
ansible-playbook playbooks/deploy.yml --ask-vault-pass    # attendu : changed=0
```

`teardown.yml` arrête et supprime les conteneurs mais **conserve les volumes** par
défaut. Il faut passer `-e supprimer_volumes=true` pour tout effacer.

---

## 9. Sécurité

- Aucun secret en clair dans le dépôt : mots de passe de base, compte SMTP et clé de
  session sont dans `vault.yml`, chiffré avec Ansible Vault. Un `.gitignore` exclut les
  fichiers d'environnement générés.
- TLS sur le `front` avec un certificat auto-signé généré par le rôle `reverse_proxy`.
- Seul le port 443 est publié sur VM-app. Base, API et Mailpit ne sont joignables que
  depuis le réseau Docker ou la boucle locale.
- SSH par clés uniquement, mot de passe désactivé, pare-feu configuré par le rôle
  `common`.

**Pas d'authentification applicative**, c'est un choix de périmètre. L'application reste
donc accessible à quiconque atteint le 443 de VM-app. L'accès est maintenu limité au
réseau de laboratoire pendant toute la durée du projet. Piste d'amélioration : une
authentification HTTP de base au niveau de nginx coûterait moins d'une heure et
fermerait cette porte sans toucher à l'application.

---

## 10. Limites connues

| Limite | Conséquence | Piste |
| --- | --- | --- |
| Pas d'authentification | accès libre sur le réseau du labo | auth basique nginx |
| Tuiles cartographiques externes | carte vide sans Internet | fond statique embarqué |
| Les deux VM sur le même disque | pas de résilience réelle | cible de sauvegarde externe |
| Certificat auto-signé | avertissement navigateur | hors périmètre, usage labo |
| Pas de tests automatisés au déploiement | régressions détectées à la main | pipeline CI, si le temps le permet |
