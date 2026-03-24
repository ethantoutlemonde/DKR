# E-Commerce Microservices - Infrastructure Docker

## Presentation du projet

Application e-commerce basee sur une architecture microservices, conteneurisee avec Docker.
Le projet comprend un frontend Vue.js et trois microservices backend (auth, product, order),
chacun avec sa propre base de donnees MongoDB.

### Architecture

```
                   +----------+
                   | Frontend |
                   | (Vue.js) |
                   +----+-----+
                        |
           +------------+------------+
           |            |            |
     +-----+----+ +----+-----+ +----+-----+
     |   Auth   | | Product  | |  Order   |
     | Service  | | Service  | | Service  |
     +-----+----+ +----+-----+ +----+-----+
           |            |            |
     +-----+----+ +----+-----+ +----+-----+
     | MongoDB  | | MongoDB  | | MongoDB  |
     |  (auth)  | |(ecommerce)| | (orders) |
     +----------+ +----------+ +----------+
```

### Composants

- **Frontend** : Interface Vue.js servie par Vite (dev) ou Nginx (prod), port 8080
- **Auth Service** : Inscription, connexion, JWT, port 3001
- **Product Service** : Produits et panier, port 3000
- **Order Service** : Commandes, port 3002
- **MongoDB** : Une instance par service


## Prerequis

### Pour le deploiement Docker (recommande)

- Docker >= 20.x
- Docker Compose >= 1.27
- Git

### Pour le deploiement manuel

- Node.js >= 18.x
- npm >= 6.x
- MongoDB 4.4
- PM2 (`npm install -g pm2`)


## Demarrage rapide

### 1. Cloner le depot

```bash
git clone <URL_DU_DEPOT>
cd e-commerce-vue
```

### 2. Lancer en mode developpement

```bash
docker-compose up --build
```

Ca lance tous les services avec le hot-reload active. Les modifications dans `src/` sont
automatiquement prises en compte grace aux volumes montes.

### 3. Initialiser les produits

Dans un autre terminal :

```bash
./scripts/init-products.sh
```

### 4. Acceder a l'application

Ouvrir http://localhost:8080 dans un navigateur.


## Configuration par environnement

### Developpement (docker-compose.yml)

Le fichier `docker-compose.yml` est configure pour le developpement :

- Les Dockerfiles ciblent le stage `development`
- Les volumes montent le code source pour le hot-reload
- Les ports sont exposes pour le debug (3000, 3001, 3002, 8080)
- Chaque service utilise `nodemon` pour redemarrer automatiquement

**Lancer :**
```bash
docker-compose up --build
```

**Arreter :**
```bash
docker-compose down
```

**Arreter et supprimer les volumes (reset des BDD) :**
```bash
docker-compose down -v
```

### Production (docker-compose.prod.yml)

Le fichier `docker-compose.prod.yml` est prevu pour un deploiement en production :

- Les images sont pre-construites et tirees depuis la registry
- Le frontend est servi par Nginx (image legere)
- Les services backend tournent sous un utilisateur non-root
- Pas de volumes de code source montes
- Limites de ressources definies (CPU, memoire)
- Compatible Docker Swarm (section `deploy` avec replicas)

**Deploiement avec Docker Compose :**
```bash
docker-compose -f docker-compose.prod.yml up -d
```

**Deploiement avec Docker Swarm :**
```bash
docker swarm init
docker stack deploy -c docker-compose.prod.yml e-commerce
```

**Verifier les services Swarm :**
```bash
docker stack services e-commerce
```


## Structure des Dockerfiles

### Services backend (auth, product, order)

Chaque service utilise un build multi-stage avec 3 etapes :

1. **base** : Installe les dependances de production uniquement (`npm ci --only=production`)
2. **development** : Installe toutes les dependances + copie le code, lance `nodemon`
3. **production** : Copie les `node_modules` depuis `base`, copie le code source, lance `node`

Bonnes pratiques appliquees :
- Image alpine pour reduire la taille
- Utilisateur non-root en production (`appuser`)
- `npm cache clean --force` apres installation
- `.dockerignore` pour exclure node_modules, tests, .env

### Frontend

Build multi-stage en 3 etapes :

1. **development** : Installe les deps, monte le code, lance `vite` (dev server avec HMR)
2. **build** : Installe les deps, compile le frontend (`npm run build`)
3. **production** : Image `nginx:alpine`, copie le build + config nginx personnalisee

En production, Nginx sert les fichiers statiques et fait le reverse-proxy vers les services backend.


## Variables d'environnement

| Variable | Description | Valeur par defaut |
|---|---|---|
| `PORT` | Port du service | 3000/3001/3002/8080 |
| `MONGODB_URI` | URI de connexion MongoDB | mongodb://mongo-xxx:27017/xxx |
| `JWT_SECRET` | Cle secrete pour les tokens JWT | XdZGHks |
| `NODE_ENV` | Environnement (development/production) | development |
| `VITE_PRODUCT_SERVICE_URL` | URL du service produit (pour le proxy) | http://product-service:3000 |
| `VITE_AUTH_SERVICE_URL` | URL du service auth (pour le proxy) | http://auth-service:3001 |
| `VITE_ORDER_SERVICE_URL` | URL du service order (pour le proxy) | http://order-service:3002 |


## Pipeline CI/CD (GitLab)

Le pipeline est defini dans `.gitlab-ci.yml` et les fichiers `build-*.yml` de chaque service.

### Stages

1. **test** : Execute les tests unitaires de chaque service
2. **build** : Construit les images Docker et les pousse vers la registry
3. **security** : Scan des images avec Trivy
4. **deploy** : (a configurer selon l'infra cible)

Les pipelines se declenchent sur les branches `develop` (tests uniquement) et `main` (tests + build + scan).

### Variables CI/CD a configurer dans GitLab

- `CI_REGISTRY_IMAGE` : URL de la registry Docker
- `CI_REGISTRY_USER` : Utilisateur de la registry
- `CI_REGISTRY_PASSWORD` : Mot de passe de la registry
- `JWT_SECRET` : Secret JWT pour la production


## Tests

### Tests backend (Jest)

```bash
cd services/auth-service && npm test
cd services/product-service && npm test
cd services/order-service && npm test
```

### Tests frontend (Vitest)

```bash
cd frontend && npm run test
```

### Lancer tous les tests

```bash
./scripts/run-tests.sh
```


## Commandes utiles

### Voir les logs d'un service

```bash
docker-compose logs -f auth-service
```

### Reconstruire un seul service

```bash
docker-compose up --build auth-service
```

### Executer une commande dans un conteneur

```bash
docker-compose exec product-service sh
```

### Tester les endpoints

```bash
# Health check
curl http://localhost:3001/api/health
curl http://localhost:3000/api/health
curl http://localhost:3002/api/health

# Inscription
curl -X POST http://localhost:3001/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "test@test.com", "password": "test123"}'

# Liste des produits
curl http://localhost:3000/api/products
```


## Bonnes pratiques appliquees

### Securite
- Conteneurs executes en tant qu'utilisateur non-root en production
- Secrets passes via variables d'environnement (pas en dur dans les images)
- Scan de vulnerabilites avec Trivy dans la CI
- Tokens JWT pour proteger les routes sensibles

### Optimisation
- Images Alpine pour reduire la taille
- Multi-stage build pour separer dev et prod
- Cache Docker optimise (COPY package*.json avant le code)
- Suppression du cache npm apres installation

### Reseau
- Reseau bridge en dev pour la communication inter-services
- Reseau overlay en prod (Docker Swarm)
- Seul le frontend expose un port vers l'exterieur en prod

### Haute disponibilite (production)
- 2 replicas par service dans Docker Swarm
- Restart automatique en cas de crash
- Limites de ressources pour eviter les OOM
