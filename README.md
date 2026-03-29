# Projet de fin de module — Gestion du département facturation (Service Immobilier)

## 1) Contexte et objectif
Cette application permet de gérer la **facturation** et les **paiements** pour deux pôles de l’entreprise :
- **Service Achat** (fournisseurs)
- **Service Commercial** (clients)

Le système doit offrir des fonctionnalités CRUD complètes, l’impression des factures, une sécurité solide via un service d’authentification externe, et des fonctions de géolocalisation (carte, distance, frais de douane).

---

## 2) Architecture globale (microservices)

### 2.1 Composants
1. **Auth Service (externe, séparé et sécurisé)**
   - Gère l’authentification/identité.
   - Fournit des jetons d’accès (ex: JWT/OAuth2).

2. **Billing Supplier Service**
   - Gestion des fournisseurs.
   - Gestion des factures fournisseurs.
   - Gestion des paiements fournisseurs.

3. **Billing Client Service**
   - Gestion des clients.
   - Gestion des factures clients.
   - Gestion des paiements clients.

4. **Geo & Customs Service (optionnel mais recommandé)**
   - Géocodage adresse → coordonnées GPS.
   - Calcul distance entre client/fournisseur et siège.
   - Détection national/étranger.
   - Calcul des frais de douane.

5. **Applications Web**
   - **Web Achat**: interface dédiée au service achat (fournisseurs + transactions).
   - **Web Commercial**: interface dédiée au service commercial (clients + transactions).

6. **API Gateway (recommandé)**
   - Point d’entrée unique.
   - Routage vers les microservices.
   - Vérification des tokens et règles d’accès.

### 2.2 Flux d’authentification (SSO)
- L’utilisateur se connecte au **Auth Service**.
- Le service renvoie un token (JWT).
- Le token donne accès automatiquement aux apps **Achat/Commercial** (SSO).
- Les microservices valident le token à chaque requête.

---

## 3) Modèle de données MySQL

### 3.1 Tables demandées

#### `fournisseurs`
- `id` (PK)
- `raison_sociale`
- `ice` / `if` / identifiant fiscal
- `email`, `telephone`
- `adresse`, `ville`, `pays`
- `latitude`, `longitude`
- `distance_km`
- `is_etranger` (bool)
- `created_at`, `updated_at`

#### `clients`
- `id` (PK)
- `nom` / `raison_sociale`
- `cin_ou_ice`
- `email`, `telephone`
- `adresse`, `ville`, `pays`
- `latitude`, `longitude`
- `distance_km`
- `is_etranger` (bool)
- `created_at`, `updated_at`

#### `factures`
- `id` (PK)
- `numero_facture` (unique)
- `type_tiers` (`CLIENT` | `FOURNISSEUR`)
- `tiers_id` (référence logique vers client/fournisseur)
- `date_facture`, `date_echeance`
- `montant_ht`, `tva`, `montant_ttc`
- `frais_douane`
- `montant_total`
- `statut` (`BROUILLON`, `VALIDEE`, `PARTIELLEMENT_PAYEE`, `PAYEE`, `ANNULEE`)
- `pdf_url` (lien vers facture imprimable)
- `created_at`, `updated_at`

#### `payement` *(ou `paiements`, orthographe recommandée)*
- `id` (PK)
- `facture_id` (FK)
- `date_paiement`
- `montant`
- `mode_paiement` (`VIREMENT`, `CHEQUE`, `ESPECES`, ...)
- `reference`
- `commentaire`
- `created_at`, `updated_at`

> Recommandation: renommer `payement` en `paiements` pour cohérence linguistique et technique.

---

## 4) Règles de gestion

1. **CRUD complet**
   - Création, consultation, modification, suppression pour clients, fournisseurs, factures, paiements.

2. **Impression des factures**
   - Génération PDF côté backend.
   - Téléchargement/aperçu depuis les apps web.

3. **Gestion des statuts facture**
   - Une facture devient `PAYEE` quand la somme des paiements atteint le total.
   - `PARTIELLEMENT_PAYEE` si paiement incomplet.

4. **Distance et géolocalisation**
   - À la création/modification d’un client/fournisseur :
     - géocoder l’adresse,
     - calculer la distance au siège,
     - enregistrer latitude/longitude + distance.

5. **Frais de douane**
   - Si `pays != Maroc` → `is_etranger = true`.
   - Ajouter automatiquement des frais de douane selon une règle configurable:
     - Ex: `frais_douane = montant_ht * taux_douane`.

6. **Sécurité / autorisation**
   - Authentification externalisée (service séparé).
   - Autorisations par rôle (`ROLE_ACHAT`, `ROLE_COMMERCIAL`, `ROLE_ADMIN`).
   - Journalisation des actions sensibles (audit).

---

## 5) Règles de sécurité techniques

- Communication HTTPS entre clients et API.
- Validation systématique du JWT.
- Protection CORS, CSRF (si session), rate limiting.
- Chiffrement des secrets (variables d’environnement / vault).
- Logs centralisés + traçabilité (request ID).
- Sauvegardes régulières de la base de données.

---

## 6) Interfaces Web

### 6.1 Application Web Achat
- Gestion fournisseurs (CRUD).
- Factures fournisseurs.
- Paiements fournisseurs.
- Vue carte des fournisseurs.
- Impression facture fournisseur.

### 6.2 Application Web Commercial
- Gestion clients (CRUD).
- Factures clients.
- Paiements clients.
- Vue carte des clients.
- Impression facture client.

### 6.3 Fonctionnalités UI communes
- Tableau de bord (KPI : montants dus/payés).
- Recherche, filtre, tri.
- Historique d’activité.

---

## 7) Exemple d’API (résumé)

### Fournisseurs / Clients
- `GET /api/fournisseurs`
- `POST /api/fournisseurs`
- `PUT /api/fournisseurs/{id}`
- `DELETE /api/fournisseurs/{id}`

- `GET /api/clients`
- `POST /api/clients`
- `PUT /api/clients/{id}`
- `DELETE /api/clients/{id}`

### Factures / Paiements
- `GET /api/factures`
- `POST /api/factures`
- `POST /api/factures/{id}/print` (génération PDF)
- `GET /api/factures/{id}/pdf`

- `GET /api/paiements`
- `POST /api/paiements`

---

## 8) Proposition de stack technique

- **Backend microservices**: Spring Boot / Node.js (NestJS) / .NET (au choix)
- **Base de données**: MySQL 8+
- **Frontend**: React / Angular / Vue
- **Cartographie**: Leaflet + OpenStreetMap (ou Google Maps)
- **Géocodage**: Nominatim / Google Geocoding API
- **Conteneurisation**: Docker + Docker Compose
- **Reverse proxy / gateway**: Nginx / Kong / Traefik

---

## 9) Plan de réalisation (sprint rapide)

1. Initialisation projet + schéma DB.
2. Développement microservice clients/fournisseurs.
3. Développement microservice factures/paiements.
4. Intégration auth externe (JWT).
5. Développement UI achat + commercial.
6. Intégration carte + distance + douane.
7. Génération PDF + tests + sécurisation.
8. Déploiement et documentation finale.

---

## 10) Livrables attendus

- Scripts SQL de création des tables.
- Code source des microservices.
- Code source des deux applications web.
- Documentation API (OpenAPI/Swagger).
- Guide d’installation et d’exploitation.
- Jeu de données de démonstration.

---

## 11) Critères de validation

- Authentification externalisée fonctionnelle (SSO).
- Toutes les entités en CRUD.
- Factures imprimables PDF.
- Gestion correcte des paiements/états des factures.
- Carte et distance visibles pour clients/fournisseurs.
- Ajout automatique des frais de douane pour l’étranger.
- Sécurité de base conforme (JWT, rôles, HTTPS, audit).
