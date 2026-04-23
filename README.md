<p align="center">
  <img src="https://raw.githubusercontent.com/traefik/traefik/master/docs/content/assets/img/traefik.logo.png" alt="Traefik" width="300"/>
</p>

<h1 align="center">Traefik v3 — Wildcard SSL avec OVH, Gandi et CrowdSec</h1>

<p align="center">
  <strong>Certificats wildcard Let's Encrypt + reverse proxy + protection anti-bot, clé en main avec Docker Compose.</strong>
</p>

<p align="center">
  <a href="#quick-start">Quick Start</a> &bull;
  <a href="#architecture">Architecture</a> &bull;
  <a href="#tutoriel-pas-à-pas">Tutoriel</a> &bull;
  <a href="#dépannage">Dépannage</a> &bull;
  <a href="#commandes-utiles">Commandes</a>
</p>

---

## Fonctionnalités

| | |
|---|---|
| **Wildcard SSL** | Certificats `*.votre-domaine.fr` automatiques via DNS-01 |
| **Multi-provider** | OVH et Gandi sur le même Traefik, simultanément |
| **Hôte simple** | Certificats classiques via TLS challenge (n'importe quel registrar) |
| **CrowdSec** | Blocage automatique des IPs malveillantes (brute force, CVE, scans) |
| **Docker-native** | Découverte automatique des conteneurs via labels |
| **Logs** | Access logs JSON + rotation quotidienne (7 jours) |
| **Prêt pour la prod** | Socket Docker read-only, tokens restreints, secrets hors du git |

---

## Quick Start

### Without Wildcard :

```bash
# 1. Cloner le dépot
git clone https://github.com/CoopCodeCommun/TraefikV3.git
cd TraefikV3

# 2. Créer le réseau Docker
docker network create frontend

# 3. Préparer les dossiers
mkdir -p letsencrypt traefik_logs
cp traefik_dynamic_exemple.yml traefik_dynamic.yml

# 4. Configurer CrowdSec (voir étape 4 du tuto)
docker compose up -d
docker exec crowdsec cscli collections install crowdsecurity/traefik
docker exec crowdsec cscli bouncers add traefik-bouncer
# → copier la clé dans traefik_dynamic.yml, puis :
docker compose restart crowdsec traefik
```

### Widlcard : 

```bash
# 1. Cloner le dépot
git clone https://github.com/CoopCodeCommun/TraefikV3.git
cd TraefikV3

# 2. Créer le réseau Docker
docker network create frontend

# 3. Préparer les dossiers
mkdir -p wildcard_conf/letsencrypt wildcard_conf/traefik_logs
cp wildcard_conf/traefik_dynamic_exemple.yml wildcard_conf/traefik_dynamic.yml

# 4. Créer le .env (voir la section "Configurer le .env" ci-dessous)
nano wildcard_conf/.env

# 5. Lancer
cd wildcard_conf && docker compose up -d

# 6. Configurer CrowdSec (voir étape 4 du tuto)
docker exec crowdsec cscli collections install crowdsecurity/traefik
docker exec crowdsec cscli bouncers add traefik-bouncer
# → copier la clé dans traefik_dynamic.yml, puis :
docker compose restart crowdsec traefik
```

---

## Architecture

```
Internet (ports 80/443)
       |
       v
   Traefik v3
       |
       +-- HTTP :80 -> redirige vers HTTPS :443
       |
       +-- HTTPS :443
       |     |
       |     +-- Plugin CrowdSec (bouncer) -> decide allow/block
       |     |
       |     +-- Routeur app-ovh    -> *.domaine-ovh.fr    (cert wildcard OVH)
       |     +-- Routeur app-gandi  -> *.domaine-gandi.fr  (cert wildcard Gandi)
       |     +-- Routeur app-simple -> hote.autre.fr       (cert TLS challenge)
       |     |
       |     +-- Service backend (whoami, votre app...)
       |
       +-- Provider Docker (decouvre les conteneurs via le socket)
       +-- Provider File  (traefik_dynamic.yml, watched)

CrowdSec (interne, pas de port expose)
       |
       +-- Lit /var/log/traefik/access.log
       +-- Analyse les patterns (brute force, CVE, scans...)
       +-- Envoie les decisions au bouncer via LAPI :8080

Logrotate
       +-- Rotation quotidienne des logs (7 jours)
```

### Resolvers ACME

| Resolver | Challenge | Wildcard | API DNS requise | Usage |
|----------|-----------|----------|-----------------|-------|
| `le-ovh` | DNS-01 | oui | OVH | `*.domaine.fr` pour domaines OVH |
| `le-gandi` | DNS-01 | oui | Gandi | `*.domaine.fr` pour domaines Gandi |
| `le-alpn` | TLS challenge | non | non | Hôte simple, n'importe quel registrar |

> **TLS challenge** (TLS-ALPN-01) : Let's Encrypt se connecte sur le port 443, Traefik répond avec un certificat temporaire de validation, et le vrai certificat est émis. Il suffit que le domaine pointe vers le serveur. Aucune API DNS n'est nécessaire, mais il ne supporte pas les wildcards.

### Contenu du dépot

```
wildcard_conf/                      # Configuration principale
  ├── docker-compose.yml            #   Stack Traefik + CrowdSec + Logrotate
  ├── traefik.yml                   #   Config statique Traefik
  ├── traefik_dynamic_exemple.yml   #   Template config dynamique (a copier)
  ├── crowdsec/config/acquis.yaml   #   Source de logs CrowdSec
  └── tests/docker-compose.yml      #   Service de test whoami
```

> Le dossier racine contient aussi une configuration **simple** (TLS challenge uniquement, pas de wildcard). Utile si vous n'avez besoin que d'un certificat par domaine.

---

## Prérequis

- Un serveur Linux avec **Docker** et **Docker Compose** installés
- Un ou plusieurs noms de domaine gérés chez **OVH** et/ou **Gandi** (pour les wildcards)
- Optionnellement, un ou plusieurs domaines hébergés ailleurs (pour les hôtes simples)
- Les **enregistrements DNS** pointant vers l'IP de votre serveur :

```dns
# Pour un wildcard (OVH ou Gandi)
votre-domaine.fr.       IN A     <IP_DU_SERVEUR>
*.votre-domaine.fr.     IN A     <IP_DU_SERVEUR>

# Pour un hôte simple (n'importe quel registrar)
sous-domaine.autre.fr.  IN A     <IP_DU_SERVEUR>
```

- Les ports **80** et **443** ouverts depuis Internet

---

## Tutoriel pas à pas

### Étape 1 — Créer les tokens API

<details>
<summary><strong>1A — Token OVH (cliquer pour déplier)</strong></summary>

Traefik utilise l'API OVH pour créer des enregistrements TXT dans votre zone DNS (challenge DNS-01).

#### Ouvrir la page de création

Remplacez `VOTRE_DOMAINE` dans l'URL ci-dessous puis ouvrez-la dans votre navigateur :

```
https://eu.api.ovh.com/createToken/?POST=/domain/zone/VOTRE_DOMAINE/*&DELETE=/domain/zone/VOTRE_DOMAINE/*
```

> **Note :** La console API OVH (https://eu.api.ovh.com/console/) fonctionne sous Chrome mais peut poser problème sous Firefox.

Connectez-vous avec vos identifiants OVH (ceux du Manager).

#### Remplir le formulaire

| Champ | Valeur |
|-------|--------|
| **Application name** | `traefik-acme-dns01` |
| **Application description** | `Traefik DNS-01 challenge for wildcard certs` |
| **Validity** | **Unlimited** |
| **Rights** | Pré-remplis via l'URL (2 lignes, voir ci-dessous) |
| **Restricted IPs** | L'IP de votre serveur (recommandé en production, vide pour un test) |

**Rights** (pré-remplis grâce à l'URL) :

| Method | Path |
|--------|------|
| `POST` | `/domain/zone/VOTRE_DOMAINE/*` |
| `DELETE` | `/domain/zone/VOTRE_DOMAINE/*` |

Ce sont les permissions **minimales** nécessaires. Traefik (via la lib Lego) découvre la zone DNS par requête SOA, il n'a pas besoin de `GET` ni de `PUT`. Le token est ainsi restreint à un seul domaine et ne peut pas toucher aux autres zones DNS de votre compte OVH.

**Restricted IPs** : en renseignant l'IP du serveur, le token ne sera utilisable que depuis cette machine. Recommandé en production.

Cliquer **"Create"**.

#### Récupérer les 3 clés

La page affiche **3 valeurs à noter immédiatement** (affichées une seule fois) :

- **Application Key** → `OVH_APPLICATION_KEY`
- **Application Secret** → `OVH_APPLICATION_SECRET`
- **Consumer Key** → `OVH_CONSUMER_KEY`

#### Points d'attention OVH

- **Validity = Unlimited** — sinon le token expire et les renouvellements automatiques de certificats échouent silencieusement.
- **Permissions restreintes au domaine** (`/domain/zone/VOTRE_DOMAINE/*`) et pas `/domain/zone/*` — un token trop large donne accès à toutes les zones DNS du compte.
- **L'endpoint doit correspondre** à votre compte : `ovh-eu` pour l'Europe (défaut), `ovh-ca` pour le Canada, etc.

#### Supprimer un token OVH existant

Si vous avez déjà créé un token trop permissif ou compromis, supprimez-le :

1. Allez sur https://eu.api.ovh.com/console/ (Chrome recommandé)
2. Connectez-vous avec vos identifiants OVH
3. Dans la section `/me/api/credential` :
   - `GET /me/api/credential` → liste des IDs de credentials
   - `GET /me/api/credential/{credentialId}` → détails pour identifier le bon
   - `DELETE /me/api/credential/{credentialId}` → révoque le Consumer Key
4. Pour supprimer l'application entièrement (Application Key + Secret + tous les Consumer Keys) :
   - `GET /me/api/application` → liste des IDs
   - `DELETE /me/api/application/{applicationId}`

</details>

<details>
<summary><strong>1B — Token Gandi (cliquer pour déplier)</strong></summary>

Gandi utilise des jetons d'accès personnel (PAT) pour l'authentification API. La variable d'environnement attendue par Traefik est `GANDIV5_PERSONAL_ACCESS_TOKEN`.

#### Ouvrir la page de création

Sur https://admin.gandi.net :
1. Menu gauche → **Organisations**
2. Sélectionner votre organisation
3. Onglet **Partage**
4. En bas → **Créer un jeton d'accès personnel**

#### Remplir le formulaire

| Champ | Valeur |
|-------|--------|
| **Nom du jeton d'accès personnel** | `traefik-acme-dns01` |
| **Expire dans** | **1 an** |
| **Ressources du jeton d'accès** | **Restreint aux produits sélectionnés** → sélectionner votre domaine |

**Permissions allouées au jeton d'accès** — activer uniquement dans la section **Domaines** :

| Permission | Activer |
|------------|---------|
| **Gérer la configuration technique des domaines** | oui |

Cela active automatiquement **Voir et renouveler les domaines** (dépendance requise). Laisser tout le reste désactivé (Organisation, Facturation, Web Hosting, Cloud, Certificats SSL).

Cliquer **Créer** puis copier le token immédiatement (affiché une seule fois).

#### Points d'attention Gandi

- **Expiration max = 1 an** — pas d'option illimitée chez Gandi. Il faudra renouveler le token avant expiration, sinon les renouvellements de certificats échoueront silencieusement. Mettez un rappel dans votre agenda.
- **Restreindre au domaine** — en choisissant "Restreint aux produits sélectionnés", le token ne peut agir que sur le(s) domaine(s) choisi(s).
- **LiveDNS requis** — le domaine doit utiliser les nameservers Gandi (`ns1.gandi.net`). C'est le cas par défaut pour les domaines Gandi récents.

#### Supprimer un jeton Gandi existant

Sur https://admin.gandi.net → **Organisations** → votre org → **Partage** → trouver le jeton dans la liste → cliquer l'icone **poubelle**.

</details>

---

### Étape 2 — Configurer le fichier `.env`

Créez le fichier `.env` dans le dossier `wildcard_conf/` :

```bash
cat > wildcard_conf/.env << 'EOF'
# OVH API (DNS-01)
OVH_APPLICATION_KEY=votre_application_key
OVH_APPLICATION_SECRET=votre_application_secret
OVH_CONSUMER_KEY=votre_consumer_key
OVH_ENDPOINT=ovh-eu

# Gandi API (DNS-01)
GANDIV5_PERSONAL_ACCESS_TOKEN=votre_token_gandi

# Domaine OVH
DOMAIN=votre-domaine-ovh.fr
DOMAIN_REGEX=^.+\.votre\-domaine\-ovh\.fr$$

# Domaine Gandi
DOMAIN_GANDI=votre-domaine-gandi.fr
DOMAIN_GANDI_REGEX=^.+\.votre\-domaine\-gandi\.fr$$

# Hôte simple (TLS challenge, pas de wildcard ni d'API DNS)
DOMAIN_SIMPLE=sous-domaine.autre-registrar.fr
EOF
```

> Si vous n'utilisez qu'un seul provider (OVH ou Gandi), laissez les variables de l'autre vides ou omettez-les. De même, `DOMAIN_SIMPLE` est optionnel.

**`DOMAIN_REGEX`** : c'est la regex qui matche tous les sous-domaines. Échappez les `.` et `-` avec `\`, et terminez par `$$` (le double `$` est l'échappement Docker Compose pour produire un `$` littéral).

| Domaine | `DOMAIN_REGEX` |
|---------|-------------|
| `example.com` | `^.+\.example\.com$$` |
| `mon-site.fr` | `^.+\.mon\-site\.fr$$` |
| `sub.domain.org` | `^.+\.sub\.domain\.org$$` |

---

### Étape 3 — Préparer l'infrastructure

```bash
# Créer le réseau Docker (si pas déjà fait)
docker network create frontend

# Créer les dossiers nécessaires
mkdir -p wildcard_conf/letsencrypt wildcard_conf/traefik_logs

# Copier le template de configuration dynamique
cp wildcard_conf/traefik_dynamic_exemple.yml wildcard_conf/traefik_dynamic.yml
```

---

### Étape 4 — Configurer CrowdSec

#### 4.1 Lancer le stack

```bash
cd wildcard_conf
docker compose up -d
```

Trois conteneurs démarrent :
- `traefik-wildcard` — le reverse proxy (ports 80/443)
- `crowdsec` — l'analyseur de logs (pas de port exposé)
- `logrotate` — rotation quotidienne des logs (7 jours de rétention)

#### 4.2 Installer la collection Traefik dans CrowdSec

```bash
docker exec crowdsec cscli collections install crowdsecurity/traefik
```

Cela installe les parsers pour les logs Traefik et les scénarios de détection (brute force, CVE, probing, etc.).

#### 4.3 Créer le bouncer et récupérer la clé API

```bash
docker exec crowdsec cscli bouncers add traefik-bouncer
```

La commande affiche une clé API — **notez-la immédiatement**, elle n'est affichée qu'une seule fois.

#### 4.4 Injecter la clé dans la configuration dynamique

Éditez `wildcard_conf/traefik_dynamic.yml` et remplacez la valeur de `crowdsecLapiKey` par la clé obtenue :

```yaml
http:
  middlewares:
    crowdsec:
      plugin:
        crowdsec-bouncer-traefik-plugin:
          enabled: true
          logLevel: INFO
          crowdsecLapiKey: VOTRE_CLE_BOUNCER_ICI
          crowdsecLapiHost: crowdsec:8080
          crowdsecAppsecEnabled: false
          crowdsecAppsecHost: crowdsec:7422
          crowdsecAppsecFailureBlock: true
          crowdsecAppsecUnreachableBlock: true
```

#### 4.5 Redémarrer le stack

```bash
docker compose restart crowdsec traefik
```

Le redémarrage est nécessaire pour que :
- CrowdSec charge les nouvelles collections
- Traefik recharge le plugin bouncer avec la bonne clé API

---

### Étape 5 — Lancer le service de test

Le dossier `tests/` contient un service `whoami` qui répond à toutes les requêtes avec les headers reçus. Les routeurs sont configurés via des labels Docker Compose.

```bash
cd tests
docker compose --env-file ../.env up -d
```

> `--env-file ../.env` est nécessaire car les labels Docker Compose utilisent les variables `DOMAIN`, `DOMAIN_REGEX`, `DOMAIN_GANDI`, `DOMAIN_GANDI_REGEX` et `DOMAIN_SIMPLE` définies dans le `.env` du dossier parent.

Vérifiez la bonne interpolation des variables :

```bash
docker compose --env-file ../.env config
```

Trois routeurs sont configurés sur le même service whoami :
- **app-ovh** — certificat wildcard via `le-ovh` (DNS-01 OVH)
- **app-gandi** — certificat wildcard via `le-gandi` (DNS-01 Gandi)
- **app-simple** — certificat hôte simple via `le-alpn` (TLS challenge)

---

### Étape 6 — Vérifier

#### Certificats

```bash
# Certificat wildcard OVH
echo | openssl s_client -connect VOTRE_DOMAINE_OVH:443 \
  -servername VOTRE_DOMAINE_OVH 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# Certificat wildcard Gandi
echo | openssl s_client -connect VOTRE_DOMAINE_GANDI:443 \
  -servername VOTRE_DOMAINE_GANDI 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# Certificat hôte simple
echo | openssl s_client -connect VOTRE_DOMAINE_SIMPLE:443 \
  -servername VOTRE_DOMAINE_SIMPLE 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```

Pour un **certificat wildcard**, vous devez voir :
```
subject=CN = votre-domaine.fr
issuer=C = US, O = Let's Encrypt, CN = R13
notBefore=...
notAfter=...
X509v3 Subject Alternative Name:
    DNS:*.votre-domaine.fr, DNS:votre-domaine.fr
```

Pour un **certificat hôte simple**, vous devez voir :
```
subject=CN = sous-domaine.autre.fr
issuer=C = US, O = Let's Encrypt, CN = R13
notBefore=...
notAfter=...
X509v3 Subject Alternative Name:
    DNS:sous-domaine.autre.fr
```

<details>
<summary><strong>Autres commandes openssl utiles</strong></summary>

```bash
# Voir la chaîne de certificats complète
echo | openssl s_client -connect VOTRE_DOMAINE:443 \
  -servername VOTRE_DOMAINE -showcerts 2>/dev/null

# Vérifier la date d'expiration uniquement
echo | openssl s_client -connect VOTRE_DOMAINE:443 \
  -servername VOTRE_DOMAINE 2>/dev/null \
  | openssl x509 -noout -dates

# Tester un sous-domaine spécifique (le certificat wildcard doit couvrir)
echo | openssl s_client -connect VOTRE_DOMAINE:443 \
  -servername test.VOTRE_DOMAINE 2>/dev/null \
  | openssl x509 -noout -subject -ext subjectAltName
```

</details>

#### Service whoami

```bash
# Domaine OVH — apex et sous-domaine
curl -sk https://votre-domaine-ovh.fr
curl -sk https://test.votre-domaine-ovh.fr

# Domaine Gandi — apex et sous-domaine
curl -sk https://votre-domaine-gandi.fr
curl -sk https://nimportequoi.votre-domaine-gandi.fr

# Hôte simple
curl -sk https://sous-domaine.autre.fr
```

Les cinq doivent répondre avec les headers whoami.

#### CrowdSec

```bash
# Vérifier que les logs sont bien lus et parsés
docker exec crowdsec cscli metrics

# Vérifier que le bouncer communique
docker exec crowdsec cscli bouncers list

# Lister les décisions (bans) — vide au départ
docker exec crowdsec cscli decisions list
```

---

## Commandes utiles

```bash
# Logs Traefik
docker logs -f traefik-wildcard

# Logs CrowdSec
docker logs -f crowdsec

# Métriques CrowdSec (vérifier la lecture des logs)
docker exec crowdsec cscli metrics

# Lister les décisions de ban
docker exec crowdsec cscli decisions list

# Bannir/débannir une IP manuellement
docker exec crowdsec cscli decisions add --ip 1.2.3.4 --duration 24h --reason "manual ban"
docker exec crowdsec cscli decisions delete --ip 1.2.3.4

# Lister les alertes
docker exec crowdsec cscli alerts list
docker exec crowdsec cscli alerts inspect -d <ID>

# Whitelister une IP (via allowlist CrowdSec)
docker exec crowdsec cscli allowlists create my_allowlist -d "my allowlist"
docker exec crowdsec cscli allowlist add my_allowlist 172.18.0.0/24

# Arrêter tout
cd wildcard_conf/tests && docker compose down
cd wildcard_conf && docker compose down
```

---

## Dépannage

<details>
<summary><strong>Le certificat n'est pas émis</strong></summary>

```bash
# Activer les logs DEBUG dans traefik.yml (décommenter log.level: DEBUG)
# puis redémarrer Traefik et consulter :
docker logs traefik-wildcard 2>&1 | grep -i acme

# Vérifier le contenu du fichier ACME :
sudo cat wildcard_conf/letsencrypt/acme.json | python3 -m json.tool
```

Causes fréquentes :
- **Mauvaises clés API** — vérifiez le `.env` (OVH : 3 clés, Gandi : 1 token PAT)
- **Mauvaise variable d'environnement Gandi** — c'est `GANDIV5_PERSONAL_ACCESS_TOKEN`, pas `GANDI_API_KEY`
- **Token trop restrictif (OVH)** — les permissions minimales sont `POST` et `DELETE` sur `/domain/zone/VOTRE_DOMAINE/*`
- **Token expiré (Gandi)** — les PAT Gandi expirent au bout d'1 an maximum
- **Endpoint OVH incorrect** — `ovh-eu` pour l'Europe, `ovh-ca` pour le Canada
- **DNS pas encore propagé** — les resolvers `1.1.1.1` et `8.8.8.8` sont configurés dans `traefik.yml` pour accélérer la vérification
- **Port 443 bloqué** — vérifiez votre firewall
- **LiveDNS non activé (Gandi)** — le domaine doit utiliser les nameservers Gandi
- **Hôte simple : DNS ne pointe pas vers le serveur** — le TLS challenge nécessite que Let's Encrypt puisse joindre le port 443 de votre serveur via le domaine

</details>

<details>
<summary><strong>CrowdSec bloque tout (erreur 403 du plugin)</strong></summary>

Le plugin CrowdSec renvoie des erreurs 403 si la clé bouncer n'est pas configurée ou invalide :
```
ERROR: CrowdsecBouncerTraefikPlugin: statusCode:403
```

Vérifiez que `crowdsecLapiKey` dans `traefik_dynamic.yml` correspond bien à la clé générée par `cscli bouncers add`.

</details>

<details>
<summary><strong>CrowdSec ne montre aucune métrique</strong></summary>

- Vérifiez que Traefik écrit bien les logs dans `/var/log/traefik` à l'intérieur du conteneur.
- Vérifiez que `crowdsec/config/acquis.yaml` pointe sur le bon chemin.

</details>

<details>
<summary><strong>404 ou mauvais routeur</strong></summary>

- Vérifiez les labels (l'orthographe compte).
- Confirmez que le conteneur est bien sur le réseau `frontend`.
- Vérifiez l'interpolation des variables : `docker compose --env-file ../.env config`

</details>

---

## Sécurité

- **Ne commitez jamais le `.env`** — il contient vos secrets API. Le `.gitignore` l'exclut déjà.
- **Token OVH restreint** — utilisez les permissions minimales (`POST` + `DELETE` sur votre domaine uniquement) et restreignez à l'IP du serveur.
- **Token Gandi restreint** — utilisez "Restreint aux produits sélectionnés" et n'activez que la permission "Gérer la configuration technique des domaines".
- **Expiration du token Gandi** — les PAT Gandi expirent au bout d'1 an maximum. Planifiez le renouvellement.
- **`acme.json`** est créé automatiquement par Traefik avec les bonnes permissions (600).
- **Mettez à jour régulièrement** Traefik, CrowdSec et le plugin bouncer.
- Le fichier `traefik_dynamic.yml` contient la cl�� bouncer CrowdSec en clair — il est exclu du git via `.gitignore`.
- Le socket Docker est monté en lecture seule (`ro`).

---

## Fichiers de configuration

| Fichier | Rôle |
|---------|------|
| `wildcard_conf/docker-compose.yml` | Stack Traefik + CrowdSec + Logrotate |
| `wildcard_conf/traefik.yml` | Config statique : entrypoints, resolvers ACME, plugin CrowdSec |
| `wildcard_conf/traefik_dynamic_exemple.yml` | Template config dynamique (copier en `traefik_dynamic.yml`) |
| `wildcard_conf/.env` | Secrets API + domaines (non commité) |
| `wildcard_conf/crowdsec/config/acquis.yaml` | Source de logs pour CrowdSec |
| `wildcard_conf/tests/docker-compose.yml` | Service de test whoami avec labels Traefik |

---

## Sources et documentation

- [Traefik v3 — Documentation officielle](https://doc.traefik.io/traefik/)
- [Lego — Provider DNS OVH](https://go-acme.github.io/lego/dns/ovh/)
- [Lego — Provider DNS Gandi v5](https://go-acme.github.io/lego/dns/gandiv5/)
- [CrowdSec — Documentation](https://docs.crowdsec.net/)
- [CrowdSec Bouncer Traefik Plugin](https://github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin)
- [OVH — Créer un token API](https://eu.api.ovh.com/createToken/)
- [Gandi — Documentation API / PAT](https://api.gandi.net/docs/authentication/)

---

## Contribuer

Les contributions sont les bienvenues. N'hésitez pas à ouvrir une issue ou une pull request.

## Licence

MIT — voir [`LICENSE`](LICENSE).

---

<p align="center">
  <sub>Maintenu par <a href="https://codecommun.coop">Coopérative Code Commun</a></sub>
</p>
