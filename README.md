# Spécifications OpenAPI — DocX & Sign (LayerOne)

Ce dossier contient les spécifications **OpenAPI 3.1** des deux API publiques de LayerOne :

| Fichier | API | Base | Opérations |
|---|---|---|---|
| [`docx-openapi.yaml`](./docx-openapi.yaml) | DocX | `https://docx.layerone.fr` | 11 |
| [`sign-openapi.yaml`](./sign-openapi.yaml) | Sign | `https://sign.layerone.fr` | 9 |

## (a) À quoi sert cette spécification

L'OpenAPI est le **contrat machine-lisible** de l'API : elle décrit chaque endpoint
(méthode, chemin, paramètres, corps de requête, réponses, schémas) et l'authentification.
C'est l'actif central qui permet de :

- **Publier l'API** sur les annuaires et marketplaces (Postman, APIs.guru, RapidAPI).
- **Générer automatiquement** des clients/SDK, de la documentation et des collections.
- **Importer l'API dans des outils no-code** (n8n, Make, Zapier) sans recoder chaque appel.
- **Garantir la cohérence** : une seule source de vérité tenue à jour avec le code.

Les deux specs utilisent le même schéma d'authentification : un en-tête HTTP
`X-API-Key` (type `apiKey`, `in: header`). **Aucune clé n'est stockée dans la spec** —
seul le mécanisme est déclaré ; l'utilisateur fournit sa propre clé générée sur
https://dev.layerone.fr.

### Source de vérité
Ces specs sont fidèles aux modules d'intégration réels :
- Make : `integrations/make/docx/modules/*.json` et `integrations/make/sign/modules/*.json`
- Zapier : `integrations/zapier/docx/index.js` et `integrations/zapier/sign/index.js`

Chaque `path` + méthode + paramètre correspond 1:1 à un module Make existant. Si l'API
évolue, mettre à jour les modules Make/Zapier **et** ces specs ensemble.

---

## (b) Comment l'utiliser pour publier

### Vérifier la validité avant toute publication

```bash
python3 -c "import yaml; yaml.safe_load(open('docx-openapi.yaml')); print('OK')"
python3 -c "import yaml; yaml.safe_load(open('sign-openapi.yaml')); print('OK')"
```

Pour une validation sémantique OpenAPI complète (recommandé avant soumission publique) :

```bash
npx @redocly/cli lint docx-openapi.yaml
npx @redocly/cli lint sign-openapi.yaml
```

### 1. Postman (import)

1. Ouvrir Postman → bouton **Import** (en haut à gauche).
2. Déposer le fichier `docx-openapi.yaml` (ou `sign-openapi.yaml`), ou coller son contenu.
3. Postman propose **« Import as an API and generate a collection »** → accepter.
4. La collection apparaît avec une requête par opération. Dans **Authorization** au niveau
   de la collection, ajouter une variable d'environnement (ex `{{apiKey}}`) injectée dans
   l'en-tête `X-API-Key`. Postman lit déjà le `securityScheme` et propose le champ.
5. Pour publier publiquement : **Publish Docs** depuis l'onglet API, ce qui génère une page
   de documentation Postman partageable.

### 2. APIs.guru (PR vers le répertoire public)

APIs.guru est un annuaire open-source d'OpenAPI. La publication se fait par **Pull Request**
vers le dépôt GitHub `APIs-guru/openapi-directory`.

Étapes précises de soumission :

1. **Fork** du dépôt `https://github.com/APIs-guru/openapi-directory`.
2. Créer l'arborescence attendue par leur convention :
   ```
   APIs/layerone.fr/docx/1.0.0/openapi.yaml
   APIs/layerone.fr/sign/1.0.0/openapi.yaml
   ```
   (chemin = `APIs/<domaine-du-provider>/<service>/<version>/openapi.yaml`)
3. Copier `docx-openapi.yaml` → `openapi.yaml` dans le dossier `docx/1.0.0/`, idem pour Sign.
4. **Compléter les champs requis par APIs.guru** dans le bloc `info` (déjà présents ou à enrichir) :
   - `info.title`, `info.version`, `info.description` ✅ déjà présents.
   - `info.contact.url` et `info.contact.email` — APIs.guru exige un contact valide.
   - `info.x-apisguru-categories` (ex `[ "office", "documents" ]`) — recommandé.
   - `info.x-logo` (URL d'un logo) — recommandé pour l'affichage dans l'annuaire.
   - `info.x-origin` — non requis (ajouté automatiquement par leur robot).
5. Valider localement avec leur outil :
   ```bash
   git clone https://github.com/APIs-guru/openapi-directory
   cd openapi-directory
   npm install
   npm run validate
   ```
6. Ouvrir une **Pull Request** depuis le fork. Le CI d'APIs.guru re-valide la spec
   (linting Spectral + cohérence). Une fois la PR mergée par les mainteneurs,
   l'API apparaît sur https://apis.guru et est servie via leur API d'agrégation.

> Note : APIs.guru référence des API **publiquement accessibles**. S'assurer que
> `docx.layerone.fr` / `sign.layerone.fr` sont joignables publiquement avant de soumettre.

### 3. RapidAPI

1. Se connecter à https://rapidapi.com/provider (RapidAPI Hub / API Studio).
2. **Add New API** → **Import from OpenAPI** → téléverser `docx-openapi.yaml`.
3. RapidAPI crée les endpoints depuis les `paths`. Vérifier que le **Base URL** pointe
   sur `https://docx.layerone.fr` (repris du bloc `servers`).
4. Dans **Security**, mapper le `securityScheme` `apiKeyHeader` sur le header `X-API-Key`.
   RapidAPI ajoute aussi ses propres en-têtes de routage (`X-RapidAPI-Key`) côté passerelle —
   conserver `X-API-Key` comme secret transmis à l'API d'origine.
5. Définir les plans tarifaires (Free/Pro…) puis **Make Public** pour publier sur le Hub.

### 4. n8n (import OpenAPI)

n8n peut consommer une API décrite en OpenAPI de deux façons :

- **Nœud HTTP Request → Import cURL / OpenAPI** : dans un workflow, ajouter un nœud
  *HTTP Request*, puis utiliser l'option d'import pour générer la requête depuis un endpoint
  de la spec. Renseigner l'authentification générique **Header Auth** avec `X-API-Key`.
- **Génération d'un nœud déclaratif custom** : les specs servent de référence pour créer
  un nœud n8n (community node) où chaque opération de la spec devient une *resource/operation*.
  Le `securityScheme` se traduit par un credential de type *API Key in Header* (`X-API-Key`).

Dans tous les cas, le fichier YAML fournit les chemins, méthodes, paramètres et corps exacts
à reproduire — il n'y a rien à deviner.

---

## Récapitulatif des opérations

### DocX (11)

| Opération | Méthode | Chemin |
|---|---|---|
| renderFacturx | POST | `/render-facturx` |
| renderDocument | POST | `/render-document` |
| listTemplates | GET | `/client/templates` |
| uploadTemplate | POST | `/client/templates` |
| downloadTemplate | GET | `/client/templates/{template_id}` |
| updateTemplate | PUT | `/client/templates/{template_id}` |
| deleteTemplate | DELETE | `/client/templates/{template_id}` |
| listTemplateVersions | GET | `/client/templates/{template_id}/versions` |
| downloadTemplateVersion | GET | `/client/templates/{template_id}/versions/{version_id}` |
| restoreTemplateVersion | POST | `/client/templates/{template_id}/restore/{version_id}` |
| getUsageStats | GET | `/usage-stats` |

### Sign (9)

| Opération | Méthode | Chemin |
|---|---|---|
| sendForSignature | POST | `/v1/documents/send` |
| detectFields | POST | `/v1/documents/detect-fields` |
| downloadSignedDocument | GET | `/v1/documents/{document_id}/download` |
| getDocumentStatus | GET | `/v1/documents/{document_id}` |
| cancelDocument | DELETE | `/v1/documents/{document_id}` |
| sendOtp | POST | `/v1/otp/request` |
| verifyOtp | POST | `/v1/otp/verify` |
| getAuditCertificate | GET | `/v1/documents/{document_id}/audit` |
| validateSignature | GET | `/v1/documents/{document_id}/validate` |

---

## Notes de fidélité

- **Content-types fidèles aux modules Make** :
  - DocX `renderFacturx` / `renderDocument` → `application/x-www-form-urlencoded`
    (modules Make `type: urlencoded`).
  - DocX `uploadTemplate` / `updateTemplate` → `multipart/form-data` (champ `template`).
  - Sign `sendForSignature`, `detectFields`, `sendOtp`, `verifyOtp` → `application/json`.
- **Réponses binaires** :
  - DocX renvoie le fichier directement (`application/pdf` ou le MIME DOCX).
  - Sign `downloadSignedDocument` renvoie un **JSON** contenant `pdf_base64`
    (le module Make décode ensuite le base64 côté client) — la spec reflète ce JSON, pas un binaire.
- **Aucune clé API en dur** : seul le `securityScheme` `apiKeyHeader` est déclaré.
- **Niveau eIDAS (Sign)** : SES (art. 25), éventuellement SES+OTP. Profil
  PAdES-B ou PAdES-B-T (TSA DFN-CERT RFC 3161). Pas d'AES/QES, pas de
  PAdES qualifiée, pas de PAdES-LTA. Cachet CA interne (`CN=LayerOne Signature`)
  sauf P12 AATL configuré — pas de confiance Adobe par défaut.
