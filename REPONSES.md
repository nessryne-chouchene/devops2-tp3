# TP3 — Réponses

** Nessryne Chouchene

> Répondez sous chaque question. Les mesures (durées, pourcentages) sont celles de **votre** machine.

## Partie 1 — Pyramide des tests
**Q1.**
- `tests/unit` : base de la pyramide (tests unitaires). Outil : Vitest. La stack Docker n'a pas besoin de tourner : le service `unit` s'exécute seul, sans réseau ni Strapi.
- `tests/component` : niveau intermédiaire (composants React). Outils : Vitest + Testing Library, avec jsdom qui simule le navigateur. Pas besoin de la stack.
- `tests/integration` : niveau d'intégration. Outil : Vitest. La stack doit tourner : le service `integration` dépend de `strapi` (healthy), donc de PostgreSQL. Le frontend n'est pas nécessaire.
- `tests/e2e` : sommet de la pyramide (bout en bout). Outil : Playwright avec Chromium. La stack complète doit tourner (frontend, Strapi, PostgreSQL), car le test pilote un vrai navigateur sur `http://frontend:3000`.

**Q2.** La base est large car les tests unitaires sont nombreux : ils sont rapides (millisecondes), peu coûteux à écrire et à maintenir, stables (aucune dépendance réseau ou base) et ils localisent précisément la cause d'une panne. Le sommet est étroit car les tests E2E sont lents (démarrage d'un navigateur et de toute la stack), coûteux à maintenir, plus fragiles (risque de tests instables) et difficiles à diagnostiquer, puisqu'un échec peut venir de n'importe quelle couche. On en garde donc peu, pour les parcours critiques.

## Partie 2 — Tests unitaires

**Q3.** 17 tests sont exécutés, 1 échoue : `formatDuration › affiche une heure pleine sans minutes`. Le message d'échec est `AssertionError: expected '1 h 0' to be '1 h'`. `Expected` est la valeur attendue par le test (`"1 h"`), `Received` est la valeur réellement renvoyée par la fonction (`"1 h 0"`).

**Q4.** Quand la durée est un multiple de 60, le reste de la division vaut 0 et l'ancien code renvoyait `"1 h 0"` au lieu de `"1 h"`, car il ajoutait toujours les minutes. La correction ajoute une condition : si `rest === 0`, on renvoie seulement les heures, sinon les heures et les minutes.

**Q5.** Avant l'ajout de tests, la couverture de `lib/cours.ts` est de 93,33 % pour les lignes, 78,57 % pour les branches et 85,71 % pour les fonctions (88,23 % pour les instructions). Le rapport HTML signale la ligne 48 comme non couverte : c'est le `return` de la recherche textuelle de `filterCours` (comparaison avec le titre et les mots-clés). Aucun test ne fournit de recherche non vide, donc la branche « `needle` n'est pas vide » de la condition de la ligne 45 n'est jamais exécutée.

**Q6.** Après les deux tests, la couverture de `lib/cours.ts` passe à 100 % pour les lignes, 100 % pour les branches et 85,71 % pour les fonctions (94,11 % pour les instructions). Le test « combine la recherche et le niveau » exécute la ligne 48 et les deux chemins de la condition de la ligne 45 ; le test sur une liste vide vérifie le cas limite. Il reste une fonction non couverte : la fonction fléchée `(topic) => normalize(topic).includes(needle)` utilisée dans `c.topics.some(...)`. Dans le test, « git » se trouve dans le titre du cours « Git et travail collaboratif » (le `||` s'arrête donc avant), et l'autre cours débutant n'a aucun mot-clé (la liste est vide, la fonction n'est jamais appelée). Il faudrait un test de recherche par mot-clé, par exemple la requête « pipeline », qui trouve le cours « Intégration et déploiement continus » uniquement par ses `topics`.

**Q7.** La couverture de lignes mesure les lignes exécutées par les tests ; la couverture de branches mesure en plus chaque chemin d'une condition (le « oui » et le « non » d'un `if` ou d'un ternaire). Un code couvert à 100 % n'est pas forcément sans bug : un test peut exécuter une ligne sans vérifier tous les cas. Exemple de ce TP : avant la correction, les tests sur 90 et 150 minutes exécutaient la ligne `return \`${hours} h ${rest}\``, qui paraissait couverte, alors que le bug sur les heures pleines (60 minutes) restait invisible.

## Partie 3 — Tests de composant

**Q8.** `CoursCard.test.tsx` vérifie ce que l'utilisateur voit réellement une fois le composant affiché : le titre est un lien vers `/cours/abc123`, la durée apparaît formatée (« 1 h 30 ») avec le libellé du niveau, la liste des mots-clés est présente ou absente selon les données. Un test unitaire de `formatDuration` vérifie seulement la chaîne renvoyée par la fonction, pas son intégration dans le composant. L'environnement `jsdom`, qui simule le navigateur, est configuré dans `frontend/vitest.config.mts` (`environment: "jsdom"`), avec `tests/setup.ts` qui charge `jest-dom` et nettoie le DOM après chaque test.

**Q9.** Un rôle ou un texte correspond à ce que perçoivent l'utilisateur et les technologies d'assistance (lecteurs d'écran) : le test reste valable si on change le style ou la structure HTML, et il oblige à écrire un HTML accessible (libellés, rôles). Une classe CSS est un détail d'implémentation : renommée ou supprimée, elle casserait le test alors que l'interface fonctionne toujours.

**Q10.** `app/page.tsx` est un composant serveur asynchrone : il appelle `connection()` et `getCours()`, c'est-à-dire qu'il interroge Strapi sur le réseau, dans un `Suspense`. Il est difficile à rendre dans jsdom, et ce qui compte est l'assemblage page, API et base de données. Ce sont les tests de bout en bout (Playwright) qui le couvrent, en utilisant le site réellement déployé.

## Partie 4 — Tests d'intégration

**Q11.** Les tests d'intégration vérifient que `lib/api.ts` fonctionne avec le **vrai** Strapi : la bonne URL, l'en-tête `X-Client-Password`, la structure JSON réellement renvoyée (champs `documentId`, `title`, `session`, `duration`, `level`, `topics`), la présence des 6 cours créés au démarrage, leur tri par séance et le renvoi de `null` pour un identifiant inexistant. Les tests unitaires et de composant travaillent avec des données fictives, sans réseau : aucun ne vérifie ce contrat avec l'API réelle.

**Q12.** Avec Strapi arrêté, les 4 tests d'intégration échouent (`4 failed`, en 17,4 s). Chaque test affiche `TypeError: fetch failed`, causé par `Error: getaddrinfo ENOTFOUND strapi` : le nom `strapi` ne peut plus être résolu par le DNS de Docker, puisque le conteneur est arrêté, et l'appel réseau de `getCours()` échoue (environ 4 s d'attente par test). Les 19 tests unitaires et de composant passent toujours, car ils n'utilisent ni réseau ni Strapi : ils travaillent avec des données fictives en mémoire. `--no-deps` empêche `docker compose run` de démarrer les services listés dans `depends_on` (Strapi et PostgreSQL) : sans cette option, Strapi serait relancé avant les tests et on ne pourrait pas observer l'échec.

**Q13.** Non. Un test d'intégration peut échouer sans bug dans le code, par exemple : Strapi arrêté ou pas encore prêt, base vide car les données de départ n'ont pas été créées ; mauvaise configuration (URL, port, réseau Docker) ; données modifiées à la main dans l'interface d'administration de Strapi ; mot de passe ou permissions de l'API changés ; lenteur ou timeout du réseau.

## Partie 5 — Tests de bout en bout

**Q14.** Le scénario qui échoue est « la fiche du cours d'introduction affiche sa durée » (`parcours.spec.ts`, ligne 27). La trace montre que le clic sur le cours, l'URL et le titre `h1` sont corrects, mais l'assertion `toBeVisible` sur le texte exact « 1 h » dans `main` échoue après 5 s d'attente : la capture de la page affiche « Durée 1 h 0 ». C'est le même bug que Q3 : `formatDuration` ajoute les minutes pour une durée de 60 minutes. Le test échoue encore alors que j'ai corrigé le code en partie 2, car le conteneur `frontend` exécute l'image construite en partie 0, avant la correction : le nouveau code n'y est qu'après `docker compose up -d --build frontend`.

**Q15.** Le même bug est détecté par un test unitaire et par un test E2E. Lequel permet de trouver la cause le plus vite ? Lequel prouve que l'utilisateur est touché ?

| Niveau | Nombre de tests | Durée d'exécution | Besoin de la stack ? |
|---|---|---|---|
| Unitaire + composant | 19 | 17,5 s (dont environ 60 % de démarrage de l'environnement jsdom) | Non |
| Intégration | 4 | 2,9 s | Oui (Strapi et PostgreSQL) |
| E2E | 4 | 12,5 s | Oui (stack complète, avec frontend) |
**Q17.** Un test « flaky » (instable) est un test qui réussit ou échoue de façon aléatoire sans que le code change, à cause de délais, du réseau, de l'ordre d'exécution ou de données partagées. Playwright réduit ce risque : l'attente automatique (`click`, `fill` attendent que l'élément soit prêt) ; les assertions qui se réessaient jusqu'au délai (`expect(...).toHaveCount`, `toHaveText`, `toBeVisible`) ; les sélecteurs par rôle, plus stables (`getByRole`) ; un contexte de navigateur neuf pour chaque test. Ici `retries: 0` est configuré : un échec n'est pas masqué par une relance automatique.

## Partie 6 — SonarQube

**Q18.** Fichier `sonar-project.properties` complété :

(colle ici le contenu de ton fichier)

- `sonar.projectKey=tp3-cours` : identifiant unique du projet dans SonarQube.
- `sonar.sources=app,components,lib` : dossiers du code applicatif à analyser.
- `sonar.tests=tests` : dossier des tests, analysés à part du code de production.
- `sonar.javascript.lcov.reportPaths=coverage/lcov.info` : rapport de couverture produit par Vitest, que SonarQube lit pour afficher la couverture.

On exclut `app/**` et `lib/api.ts` du calcul de couverture, car ils ne sont pas testés par les tests unitaires et de composant (ceux qui produisent le rapport lcov) : `api.ts` est testé par les tests d'intégration, et les pages `app/**` par les tests de bout en bout. Les inclure ferait afficher une couverture de 0 % pour du code en réalité testé à un autre niveau, ce qui fausserait le Quality Gate.

**Q19.** Analyse du code d'origine avec le Quality Gate TP3 : le statut est **Failed**. La condition en échec est « Vulnerabilities is greater than 0 » sur le code global (1 vulnérabilité). La condition de couverture (moins de 90 %) est respectée avec 97,8 %. Résultats : 0 bug, 1 vulnérabilité, aucun security hotspot, 14 code smells, couverture de 97,8 %, duplication de 5,6 %, pour 778 lignes de code.

**Q20.** Un **bug** est une erreur de code qui provoque ou peut provoquer un comportement incorrect à l'exécution (ici : aucun, la note de fiabilité est A). Une **vulnérabilité** est une faille de sécurité qu'un attaquant peut exploiter : dans mon rapport, c'est le mot de passe écrit en dur dans `lib/api.ts`. Un **security hotspot** est un code sensible pour la sécurité qui demande une revue humaine pour décider s'il y a un risque réel (aucun dans mon rapport). Un **code smell** est un problème de maintenabilité qui n'empêche pas le code de fonctionner mais le rend plus difficile à lire ou à modifier : dans mon rapport, par exemple, les imports inutilisés et les `if` imbriqués dans `CoursList.tsx`.

**Q21.** La vulnérabilité est signalée par la règle de SonarQube sur les **identifiants et mots de passe écrits en dur** (« Hard-coded credentials », sévérité *major*) : le mot de passe `Tp3-Cours-2026!` était une constante dans `frontend/lib/api.ts`. Le risque est que toute personne qui lit le code source, le dépôt Git ou l'image Docker obtient le secret. Dans la correction fournie, le mot de passe vient de l'environnement : il est défini dans `.env` (non commité), le `docker-compose.yml` le transmet au conteneur `frontend` avec `STRAPI_CLIENT_PASSWORD: ${STRAPI_CLIENT_PASSWORD}`, et le code le lit avec `process.env.STRAPI_CLIENT_PASSWORD`. Un secret dans le code est dangereux même dans un dépôt privé : il est accessible à tous ceux qui ont accès au dépôt, il reste dans l'historique Git même après suppression, il se retrouve dans les images et les sauvegardes, et on ne peut pas le changer sans modifier et redéployer le code. Un dépôt privé peut aussi devenir public ou être divulgué.

**Q22.** La correction de `CoursList.tsx` supprime des problèmes de maintenabilité de sévérité mineure :
- **Imports inutilisés** (règle « Unused imports should be removed », *minor*) : `useEffect` et `sortBySession` étaient importés sans être utilisés. Le nouvel en-tête ne conserve que ce qui sert.
- **`if` seul dans un `else`** (règle « 'If' statement should not be the only statement in 'else' block », *minor*) : l'ancienne fonction `describeResults` imbriquait des `else { if (...) }`, ce qui rendait la logique difficile à lire. Le nouveau code sépare la logique en deux petites fonctions (`describeEmpty` et `describeResults`), avec des retours anticipés et des expressions conditionnelles : il n'y a plus d'imbrication de `else`.
Au total, le nombre de code smells passe de 14 à 9.

**Q23.** Le code dupliqué (5,7 % des lignes) se trouve, d'après le code, entre `components/CoursCard.tsx` et la page `app/cours/[documentId]/page.tsx` : les deux contiennent le même bloc `<dl className="cours-meta">` (séance, durée, niveau) et la même liste de mots-clés. Pour le supprimer, on extrairait un composant partagé, par exemple `CoursMeta`, qui affiche les métadonnées et les mots-clés, et on l'utiliserait dans les deux fichiers.

**Q24.** La dette technique est le temps estimé nécessaire pour corriger tous les problèmes de maintenabilité (code smells) du projet. SonarQube l'exprime en durée (minutes, heures, jours ; un jour vaut 8 heures), en additionnant l'effort de remédiation de chaque issue (par exemple « 5 min effort » pour chaque problème de props). Valeur avant les corrections : **… (à compléter)** pour 14 code smells. Valeur après : **… (à compléter)** pour 9 code smells.
**Q25.** Le « New Code » est le code ajouté ou modifié depuis un point de référence (par exemple la version précédente ou les N derniers jours). Les conditions du gate TP3 qui portent sur le nouveau code sont : *Security Hotspots Reviewed* inférieur à 100 %, *Coverage* inférieure à 80 % et *Duplicated Lines* supérieur à 3 % (conditions ajoutées d'office par SonarQube). SonarQube recommande de juger le nouveau code plutôt que le code global car on ne peut pas corriger tout l'héritage d'un coup : on s'assure que chaque nouvelle modification est propre (principe « Clean as You Code »), ce qui empêche la dette d'augmenter et garde le gate réaliste et actionnable.
**Q26.** Avec `sonar.qualitygate.wait=true`, que fait le scanner quand le Quality Gate est rouge ? Pourquoi est-ce utile dans un pipeline CI ?

## Partie 7 — Synthèse

**Q27.** Proposez l'ordre des étapes d'un pipeline CI pour ce projet (build, tests unitaires, intégration, E2E, analyse Sonar…) et justifiez-le. Quelles étapes doivent bloquer le merge d'une pull request ?
