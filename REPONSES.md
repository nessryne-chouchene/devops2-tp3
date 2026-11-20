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
**Q17.** Qu'est-ce qu'un test « flaky » (instable) ? Quels mécanismes de Playwright utilisés ici (attente automatique, assertions `expect(...).toHaveCount`, etc.) réduisent ce risque ?

## Partie 6 — SonarQube

**Q18.** Collez votre `sonar-project.properties` complété et expliquez chaque valeur ajoutée. Pourquoi exclut-on `app/**` et `lib/api.ts` du calcul de couverture ?

**Q19.** Analyse avec le Quality Gate TP3 : statut du gate et conditions en échec ; nombre de bugs, vulnérabilités, security hotspots et code smells ; couverture ; taux de duplication.

**Q20.** Expliquez la différence entre bug, vulnérabilité, security hotspot et code smell, avec un exemple tiré de votre rapport quand il en existe.

**Q21.** Ouvrez la vulnérabilité : quelle règle est en cause et quel est le risque ? Expliquez la correction fournie : d'où vient maintenant le mot de passe, et par quel chemin arrive-t-il jusqu'au code ? Pourquoi un secret dans le code est-il dangereux, même dans un dépôt privé ?

**Q22.** Quels code smells la correction fournie de `CoursList.tsx` fait-elle disparaître ? Pour chacun : règle, sévérité, explication, et ce qui, dans le nouveau code, règle le problème.

**Q23.** Où se trouve le code dupliqué ? Comment le supprimer ? (Ne le faites pas, décrivez seulement.)

**Q24.** Qu'est-ce que la dette technique dans SonarQube et comment est-elle exprimée ? Valeur avant et après vos corrections.

**Q25.** Que signifie « New Code » dans SonarQube ? Quelles conditions du gate TP3 portent sur le nouveau code ? Pourquoi SonarQube recommande-t-il de juger le nouveau code plutôt que le code global ?

**Q26.** Avec `sonar.qualitygate.wait=true`, que fait le scanner quand le Quality Gate est rouge ? Pourquoi est-ce utile dans un pipeline CI ?

## Partie 7 — Synthèse

**Q27.** Proposez l'ordre des étapes d'un pipeline CI pour ce projet (build, tests unitaires, intégration, E2E, analyse Sonar…) et justifiez-le. Quelles étapes doivent bloquer le merge d'une pull request ?
