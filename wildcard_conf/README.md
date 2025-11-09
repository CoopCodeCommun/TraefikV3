# wildcard_conf — Certificats wildcard OVH + Gandi et hôte simple

Ce dossier propose une configuration Traefik v3 prête à l’emploi pour :
- Certificat wildcard via OVH pour `*.demo-tibillet.ovh`.
- Certificat wildcard via Gandi pour `*.codecommun.coop`.
- Certificat classique (hôte unique) pour `agenda.nasjo.fr` (TLS-ALPN).

Les certificats wildcard nécessitent obligatoirement le challenge ACME `DNS-01`.

---

## Contenu
- `docker-compose.yml` — lance uniquement Traefik (pas de conteneur de test).
- `traefik.yml` — configuration statique (entrypoints, resolvers ACME, plugin CrowdSec, middleware global).
- `traefik_dynamic.yml` — middleware CrowdSec (`crowdsec@file`).
- `tests/docker-compose.yml` — conteneurs de test (whoami) avec trois routeurs (OVH, Gandi, agenda) et middleware CrowdSec.
- `letsencrypt/acme.json` — stockage ACME (sera créé/monté par vous).

Traefik est exposé en 80/443. Les routeurs de test sont maintenant définis par labels sur les services `whoami` dans `tests/docker-compose.yml` :
- `whoami-ovh` → matche `sub.demo-tibillet.ovh` (un seul niveau) et demande un wildcard via OVH.
- `whoami-gandi` → matche `sub.codecommun.coop` (un seul niveau) et demande un wildcard via Gandi.
- `whoami-agenda` → matche `agenda.nasjo.fr` via challenge TLS-ALPN.

Si vous souhaitez autoriser plusieurs niveaux de sous-domaines, remplacez `{sub:[^.]+}` par `{sub:.+}` dans les règles `HostRegexp`.

---

## Prérequis
- Docker et Docker Compose.
- Réseau Docker `frontend` existant (externe).
- Les zones DNS pointent vers ce serveur :
  - `*.demo-tibillet.ovh` gérée chez OVH (pas besoin d’A pour wildcard, mais vous aurez des TXT créés par le challenge). Vos sous-domaines réels doivent pointer (A/AAAA/CNAME) vers l’IP de ce Traefik.
  - `*.codecommun.coop` gérée chez Gandi (idem).
  - `agenda.nasjo.fr` (enregistrement A/AAAA vers l’IP de ce serveur).
- Ports 80 et 443 ouverts depuis Internet.

---

## Variables d’environnement (fichier `.env`)
Créez un fichier `.env` à côté de `docker-compose.yml` et ajoutez :
```
# OVH
OVH_APPLICATION_KEY=xxxxxxxx
OVH_APPLICATION_SECRET=xxxxxxxx
OVH_CONSUMER_KEY=xxxxxxxx
# optionnel, défaut ovh-eu (EU). Autres: ovh-ca (Canada), ovh-us (USA), soyoustart-eu, soyoustart-ca, kimsufi-eu, kimsufi-ca
OVH_ENDPOINT=ovh-eu

# Gandi
GANDI_API_KEY=xxxxxxxx
```
Ces variables sont utilisées par lego (la lib ACME intégrée à Traefik) pour piloter le DNS-01.

---

## Générer les clés/jetons

### 1) OVH — Application Key/Secret et Consumer Key
Référence: https://api.ovh.com/

A. Créer l’Application (AK/AS)
1. Allez sur https://api.ovh.com/createApp/
2. Donnez un nom (ex: "Traefik DNS-01") et une description.
3. Validez → vous obtenez `Application Key` (AK) et `Application Secret` (AS).

B. Créer la Consumer Key (CK) avec permissions DNS
Méthode la plus simple via l’API Explorer:
1. Allez sur https://api.ovh.com/console/
2. En haut, cliquez sur "Login" (authentifiez-vous).
3. Dans la barre de recherche, tapez `POST /auth/credential`.
4. Cliquez sur l’endpoint, puis "Try".
5. Renseignez le champ `accessRules` avec les droits nécessaires pour votre zone:
   Exemple de règles minimales pour gérer les enregistrements DNS de vos zones:
   ```json
   [
     {"method": "GET", "path": "/domain/zone/*"},
     {"method": "POST", "path": "/domain/zone/*"},
     {"method": "PUT", "path": "/domain/zone/*"},
     {"method": "DELETE", "path": "/domain/zone/*"}
   ]
   ```
   Vous pouvez restreindre à une zone précise, ex: `/domain/zone/demo-tibillet.ovh/*`.
6. Cliquez sur "Execute" → une URL de validation s’affiche.
7. Ouvrez cette URL et validez la demande pour générer la `consumerKey`.
8. Vous avez maintenant AK, AS et CK → placez-les dans `.env`.

Notes:
- `OVH_ENDPOINT` selon votre compte/zone: `ovh-eu` (Europe), `ovh-ca` (Canada), `ovh-us` (USA), etc.
- Évitez de donner des droits trop larges; restreignez au strict nécessaire.

### 2) Gandi — API Key LiveDNS
Référence: https://api.gandi.net/

1. Connectez-vous à votre compte Gandi (https://account.gandi.net/).
2. Allez dans "Sécurité" → "Tokens API" (ou "API" → "Create a personal access token" selon interface).
3. Créez un nouveau token pour LiveDNS (API v5). Donnez-lui un nom (ex: "Traefik DNS-01").
4. Copiez la clé affichée → c’est `GANDI_API_KEY`.
5. Assurez-vous que les domaines visés utilisent LiveDNS chez Gandi (pas un DNS externe).
6. Collez la clé dans `.env`.

Sécurité:
- Traitez ces secrets comme des mots de passe. Ne les commitez pas.
- Vous pouvez utiliser Docker secrets si besoin.

---

## Démarrage
1. Créez les dossiers et le fichier ACME (permissions 600 conseillées):
```bash
mkdir -p wildcard_conf/letsencrypt wildcard_conf/traefik_logs
[ -f wildcard_conf/letsencrypt/acme.json ] || (touch wildcard_conf/letsencrypt/acme.json && chmod 600 wildcard_conf/letsencrypt/acme.json)
```
2. Créez le réseau si absent:
```bash
docker network create frontend || true
```
3. Lancez:
```bash
cd wildcard_conf
# Assurez-vous d’avoir .env avec les clés
docker compose up -d
```
4. Surveillez les logs pour l’émission des certificats:
```bash
docker logs -f traefik-wildcard
```

---

## Ce que fait chaque resolver
- `le-ovh` (DNS-01): émet un certificat incluant `demo-tibillet.ovh` (apex) et `*.demo-tibillet.ovh` via les labels du routeur `app-ovh`.
- `le-gandi` (DNS-01): idem pour `codecommun.coop` et `*.codecommun.coop` via `app-gandi`.
- `le-alpn` (TLS-ALPN): émet un certificat pour `agenda.nasjo.fr`.

Remarques importantes:
- Un certificat `*.domaine.tld` ne couvre pas `domaine.tld` (apex). C’est pourquoi les labels incluent `main` (apex) + `sans` (wildcard), afin de couvrir les deux.
- Pour que `agenda.nasjo.fr` fonctionne, un enregistrement A/AAAA doit pointer vers l’IP publique de ce serveur.

---

## Adapter pour votre appli (Nginx → Django)
- Les trois routeurs pointent sur le même service `nginx` (port 80). Remplacez-le par votre conteneur Nginx qui reverse‑proxy vers Django.
- Exemple: remplacez `image: nginx:alpine` par votre propre image et vos volumes de conf.
- Dans Django, ajoutez les hôtes autorisés:
```python
ALLOWED_HOSTS = [
    ".demo-tibillet.ovh",  # permet *.demo-tibillet.ovh
    ".codecommun.coop",    # permet *.codecommun.coop
    "agenda.nasjo.fr",
]
```

---

## Dépannage
- `could not obtain certificates` (ACME):
  - Vérifiez les variables `.env` (mauvaises clés, endpoint OVH erroné, etc.).
  - OVH: la CK doit avoir les droits sur la zone et être validée.
  - Gandi: utilisez bien LiveDNS et le bon token.
  - Attendez la propagation DNS des enregistrements TXT (quelques dizaines de secondes), éventuellement ajoutez des `dnsChallenge.resolvers` publics (déjà présent).
- Wildcard OK mais apex échoue:
  - Assurez-vous que `main` est bien `domaine.tld` et que `sans` contient `*.domaine.tld`.
- L’hôte ne matche pas:
  - Vérifiez la regex `HostRegexp`. Pour plusieurs niveaux de sous-domaines, remplacez `[^.]+` par `.+`.
- Certificat pour `agenda.nasjo.fr` échoue:
  - Vérifiez que l’enregistrement DNS A/AAAA pointe vers l’IP du serveur et que le port 443 est ouvert.

---

## Sécurité
- Ne commitez jamais `.env` avec des secrets.
- `letsencrypt/acme.json` doit idéalement avoir `chmod 600`.
- Mettez à jour Traefik vers la dernière version stable lorsque possible.
