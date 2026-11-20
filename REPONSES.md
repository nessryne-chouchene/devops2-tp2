# Réponses au TP2 — Docker Compose

## Partie 1

**Q1.** Strapi attend les variables `DATABASE_CLIENT`, `DATABASE_HOST`, `DATABASE_PORT`, `DATABASE_NAME`, `DATABASE_USERNAME` et `DATABASE_PASSWORD` (plus, en option, `DATABASE_URL`, `DATABASE_SSL` et `DATABASE_SCHEMA`). `DATABASE_HOST` doit valoir `postgres`, le nom du service dans `docker-compose.yml`, que le DNS interne de Docker résout vers le conteneur PostgreSQL. `localhost` ne convient pas : dans le conteneur Strapi, il désigne le conteneur Strapi lui-même, où aucune base ne tourne.

**Q2.** `backend/.env.example` est un modèle qui liste les variables attendues, avec des valeurs d'exemple. Il est commité pour que chacun puisse créer son propre `.env`. Le vrai `.env` contient les secrets (mots de passe, clés, sels). Il ne doit jamais être commité : le dépôt est public et l'historique Git est permanent, donc un secret commité est considéré comme compromis.

## Partie 2

**Q3.** `docker compose down` arrête et supprime les conteneurs et les réseaux, mais conserve les volumes : les données de la base survivent. `docker compose down -v` supprime aussi les volumes nommés : la base est perdue et sera recréée vide au prochain démarrage.

**Q4.** Compose interprète `${...}` dans le fichier YAML. Avec `$${POSTGRES_USER}`, `$$` est un dollar échappé : Compose ne remplace rien et transmet le texte littéral `${POSTGRES_USER}` au conteneur. C'est alors le shell du conteneur (`CMD-SHELL`) qui lit la variable d'environnement du conteneur.

## Partie 3

**Q5.** `"Host": "postgres"` : pgAdmin tourne dans son propre conteneur, où `localhost` le désignerait lui-même. Il joint la base par le nom du service, résolu par le DNS de Docker. `:ro` signifie *read-only* : le fichier `servers.json` est monté en lecture seule, le conteneur ne peut pas le modifier. `${PGADMIN_PORT:-5050}` vaut la valeur de la variable `PGADMIN_PORT`, ou `5050` si elle est absente ou vide.

## Partie 4

**Q6.** Docker met chaque instruction en cache sous forme de couche. En copiant d'abord `package*.json` puis en lançant `npm ci`, la couche des dépendances est réutilisée tant que les dépendances ne changent pas. Une modification du code n'invalide que les couches suivantes. Après la modification d'un fichier de `backend/config/`, `COPY . .` et `npm run build` sont rejoués, alors que `FROM`, `WORKDIR`, `COPY package*.json` et `npm ci` viennent du cache.

**Q7.** `USER node` exécute le processus avec un utilisateur non privilégié (principe du moindre privilège) : si l'application est compromise, l'attaquant n'a pas les droits root dans le conteneur. `--chown=node:node` donne la propriété des fichiers copiés à cet utilisateur ; sinon ils appartiendraient à root et `node` ne pourrait pas écrire dans les dossiers dont Strapi a besoin (cache, uploads).

**Q8.** `depends_on` avec `condition: service_healthy` fait attendre à Compose que le healthcheck de `postgres` soit réussi avant de démarrer `strapi`. Sans cela, Strapi démarrerait dès que le conteneur PostgreSQL est créé, alors que la base n'accepte pas encore de connexions : il échouerait à se connecter et planterait ou redémarrerait en boucle.

**Q9.** Dans la table `articles`, chaque article publié occupe 2 lignes : Strapi 5 stocke une version brouillon (draft) et une version publiée.

## Partie 5

**Q10.** Le code du frontend s'exécute dans le conteneur `frontend`. Là, `localhost:1337` désigne le conteneur frontend lui-même, où rien n'écoute sur ce port. Strapi est dans un autre conteneur : il faut utiliser son nom de service, `http://strapi:1337`.

**Q11.** Le Dockerfile a 3 étapes (`FROM`) : `deps`, `builder` et `runner`. L'image finale ne contient que la dernière étape : l'image de base `node:22-alpine`, le dossier `public`, le serveur minimal `.next/standalone` (avec seulement les dépendances nécessaires) et `.next/static`. Elle ne contient ni les sources, ni les devDependencies, ni le `node_modules` complet. Sans `output: "standalone"`, le dossier `.next/standalone` n'est pas généré : le `COPY --from=builder /app/.next/standalone` échoue (fichier introuvable) et le build s'arrête. Taille de l'image `tp2/frontend` : 334 Mo sur le disque (85,7 Mo de contenu).

## Partie 6

**Q12.** `strapi` est dans les deux réseaux parce qu'il fait le lien : le frontend le joint via le réseau `frontend`, et lui joint la base via le réseau `backend`. Le frontend n'a ainsi aucun accès direct à PostgreSQL (vérifié : `nc postgres 5432` depuis le frontend renvoie `bad address`). Ne publier aucun port pour `postgres` le rend inaccessible depuis la machine hôte et depuis l'extérieur : seuls les conteneurs du réseau `backend` (`strapi`, `pgadmin`) peuvent l'atteindre, ce qui réduit la surface d'attaque.

**Q13.** Deux mesures de durcissement en production :
- placer un reverse proxy HTTPS devant les services, ne pas exposer pgAdmin publiquement (ou le limiter à `127.0.0.1`), et régénérer des mots de passe et des secrets forts, stockés via Docker secrets ou un gestionnaire de secrets plutôt que dans un `.env` ;
- épingler les images par version ou digest et les scanner, exécuter les conteneurs avec un système de fichiers en lecture seule, sans privilèges inutiles (`cap_drop`), avec des limites de ressources.

## Partie 7

Persistance : après `docker compose down` puis `docker compose up -d`, les articles sont toujours présents, car le volume `pgdata` est conservé. Avec `down -v`, ils disparaissent avec le volume.

Panne de Strapi : après `docker compose stop strapi`, le site reste en ligne (le conteneur `frontend` tourne toujours) mais n'affiche plus d'articles. Il montre à la place le message « Impossible de joindre Strapi : TypeError: fetch failed » : l'appel serveur vers `http://strapi:1337` échoue, car le service est arrêté. Les articles réapparaissent après `docker compose start strapi`, sans reconstruire quoi que ce soit. Le site reprend normalement après `docker compose start strapi`.

**Q14.** `restart: unless-stopped` fait redémarrer automatiquement le conteneur s'il s'arrête tout seul (crash, code de sortie non nul), ainsi qu'au redémarrage du démon Docker ou de la machine, sauf s'il a été arrêté manuellement. Après `docker compose stop strapi` ou `docker compose kill strapi`, l'arrêt est volontaire : Docker ne le relance donc pas. Il le redémarre automatiquement après un plantage du processus ou un redémarrage de Docker, si le conteneur n'avait pas été arrêté à la main.