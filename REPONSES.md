# TP4 — Réponses

**Nom Prénom :Nour Laachaal

> Répondez sous chaque question. Si vous avez utilisé `scripts/rattrapage-npm.sh`, indiquez-le ici :

## Partie 1 — Les dépôts npm

**Q1.** En une phrase chacun : à quoi servent `npm-proxy`, `npm-hosted` et `npm-group` ? Lequel des trois les clients (npm, le build Docker) connaissent-ils ?
Un dépôt proxy met en cache un registre distant (npmjs.org), un dépôt hosted stocke les artefacts produits en interne, et un dépôt group les regroupe pour offrir un point d'entrée unique aux clients.
**Q2.** Pourquoi l'ordre des membres de `npm-group` compte-t-il ? Que se passerait-il si un paquet `@devops2/cours-utils` était aussi publié sur npmjs.org ?
La politique Disable redeploy empêche d'écraser une version déjà publiée, garantissant l'immutabilité des artefacts et la reproductibilité des builds.
**Q3.** Pourquoi publier avec l'utilisateur `deployer` plutôt qu'avec `admin` ? Pourquoi active-t-on la lecture anonyme ?
On utilise un compte dédié (deployer) avec des droits minimaux (principe du moindre privilège) plutôt que le compte admin pour limiter les risques en cas de compromission des identifiants CI/CD.
## Partie 2 — Consommer via le group

**Q4.** Après le `npm ci`, que contient `npm-proxy` ? Pourquoi le `package-lock.json` n'a-t-il pas changé alors que les paquets viennent désormais de Nexus ?
 On utilise nexus et non localhost car la commande s'exécute dans un conteneur Docker. Dans le réseau Docker Compose, nexus est le nom d'hôte résolu par le DNS interne.
**Q5.** Demain, npmjs.org est indisponible. Que se passe-t-il au prochain `npm ci` du front ? Et pour un paquet que personne n'a encore téléchargé via votre Nexus ?
Le package-lock.json ne change pas (ou peu) car les URLs des registres dans le lockfile peuvent rester relatives ou les hashes des paquets restent identiques, garantissant que l'arbre de dépendances est le même.
**Q6.** Pourquoi `frontend/.npmrc` est-il commité ? Pourquoi le build Docker du front n'a-t-il besoin d'aucun secret ?
Le extra_hosts: "nexus:host-gateway" dans le docker-compose.yml permet au conteneur de build (qui n'est pas dans le réseau Compose par défaut) de résoudre nexus vers l'adresse IP de la machine hôte (votre PC Windows).
## Partie 3 — Publier un paquet interne

**Q7.** À quoi servent les champs `files` et `publishConfig` ? Que se passerait-il au `npm publish` sans `publishConfig` ?
Le scope @devops2 permet d'éviter les collisions de noms avec des paquets publics sur npmjs.org.
**Q8.** Recopiez le message obtenu à la seconde publication de 0.1.0. Pourquoi Nexus refuse-t-il ? Quel problème cela évite-t-il pour ceux qui ont déjà installé 0.1.0 (pensez au lockfile) ?
publishConfig force npm publish à cibler le dépôt hosted (écriture), tandis que le .npmrc de base cible le group (lecture).

**Q9.** Pourquoi la version 0.2.0, et pas 0.1.1 ni 1.0.0 ?
 La nouvelle version doit être 0.2.0 (incrément du chiffre mineur) car on ajoute une fonctionnalité (formatLevel) sans casser la compatibilité descendante (règle SemVer).
**Q10.** Quelles versions la plage `^0.2.0` accepte-t-elle ? Si vous publiez 0.2.1 puis 0.3.0, lesquelles le front prendra-t-il au prochain `npm install` ? Et au prochain `npm ci` ?
La republication échoue car la politique Disable redeploy du dépôt hosted interdit d'écraser une version existante, garantissant l'intégrité du cache.
**Q11.** Où se trouve `NPM_AUTH` et pourquoi n'apparaît-il jamais dans Git ? Que faut-il faire si un jeton a été commité par erreur ?
On supprime la copie locale pour éviter la duplication de code et s'assurer que le frontend utilise bien la version centralisée et testée du paquet.
## Partie 4 — L'image Docker

**Q12.** Décomposez `localhost:8082/cours-frontend:1.0.0` (registre, dépôt, tag). Pourquoi pousse-t-on sur le port 8082 et tire-t-on sur le port 8083 ?
 On pousse (push) sur le port 8082 (dépôt hosted dédié à l'écriture interne) et on tire (pull) sur le port 8083 (dépôt group qui agrège hosted et proxy pour la lecture).
**Q13.** Recopiez le digest obtenu au push et celui obtenu au pull. Sont-ils identiques ? Que garantit le digest que le tag ne garantit pas ?
 Un tag (ex: 1.0.0) est un alias mutable qui peut être déplacé, tandis qu'un digest (ex: sha256:...) est une empreinte cryptographique immuable du contenu exact de l'image.
**Q14.** Pourquoi ne publie-t-on jamais une image sous le seul tag `latest` pour la déployer ?
Tirer postgres via le group (8083) permet de mettre en cache l'image publique dans Nexus, évitant ainsi les limites de taux (rate limits) de Docker Hub et accélérant les futurs déploiements.
**Q15.** Pourquoi Docker accepte-t-il ici de parler à Nexus en HTTP ? Que faudrait-il pour un Nexus partagé par toute une équipe ?
Supprimer le bloc build: force Docker Compose à utiliser l'image pré-construite et poussée sur Nexus, garantissant que l'environnement de production utilise exactement le même artefact que celui testé.
## Partie 5 — Synthèse

**Q16.** « Build once, deploy many » : pour l'image du front, qu'est-ce qui est construit une seule fois, où est-ce stocké, et comment le `docker-compose.yml` le déploie-t-il ? Citez deux étapes de ce TP qu'un serveur d'intégration continue (séance 6) devrait automatiser.
Le fichier .env est exclu du versionnement via .gitignore pour éviter d'exposer des secrets. Dans .npmrc, on utilise une variable d'environnement ${NPM_AUTH} plutôt qu'une valeur en dur, ce qui garantit qu'aucun secret n'est commité dans l'historique Git, même en cas de fuite du code source.