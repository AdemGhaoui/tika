# IFT3913 — Tâche 2 : augmentation d'une suite de tests

## Membres du binôme

- `[Georges Divine Cacko Mouguen — 20386389]`
- `[Adem-Abdelkrim Ghaoui — 20362049]`

## Dépôt et cas d'étude

- **Dépôt :** <https://github.com/ReddKingK17/tika>
- **Projet :** Apache Tika 4.0.0-SNAPSHOT
- **Module :** `tika-serialization`
- **Classes étudiées :**
  - `org.apache.tika.serialization.ComponentNameResolver`
  - `org.apache.tika.config.loader.ComponentRegistry`

Les fichiers de tests ajoutés sont :

- [`ComponentNameResolverChatUniTestTest.java`](tika-serialization/src/test/java/org/apache/tika/serialization/ComponentNameResolverChatUniTestTest.java);
- [`ComponentRegistryMutationTest.java`](tika-serialization/src/test/java/org/apache/tika/config/loader/ComponentRegistryMutationTest.java).

La configuration de ChatUniTest et de PIT se trouve dans
[`tika-serialization/pom.xml`](tika-serialization/pom.xml).

La configuration ChatUniTest utilisée pour `ComponentNameResolver` était la suivante :

```xml
<plugin>
  <groupId>io.github.zju-aces-ise</groupId>
  <artifactId>chatunitest-maven-plugin</artifactId>
  <version>2.1.1</version>
  <configuration>
    <apiKeys>ollama</apiKeys>
    <model>code-llama</model>
    <url>http://localhost:11434/v1/chat/completions</url>
    <testNumber>1</testNumber>
    <maxRounds>2</maxRounds>
    <thread>false</thread>
  </configuration>
</plugin>
```

La génération ciblée a été lancée depuis `tika-serialization` avec la commande suivante :

```powershell
..\mvnw.cmd io.github.zju-aces-ise:chatunitest-maven-plugin:2.1.1:method `
  "-DselectMethod=org.apache.tika.serialization.ComponentNameResolver#usesCompactFormat"
```

## 1. Choix des classes

Les deux classes avaient déjà des tests, mais leur comportement n'était pas entièrement couvert et plusieurs mutants PIT n'étaient pas détectés.

### 1.1 `ComponentNameResolver`

La suite originale contenait `ComponentNameResolverTest`. Elle vérifiait principalement qu'un nom de composant inconnu produisait un message utile. L'analyse PIT initiale a confirmé que la majorité des responsabilités de la classe n'étaient pas exercées.

| Mesure initiale | Valeur |
|---|---:|
| Couverture de lignes PIT | 43/119 (36 %) |
| Mutants générés | 39 |
| Mutants tués | 1/39 (3 %) |
| Mutants sans couverture | 37 |
| Mutants survivants | 1 |
| Test strength | 50 % |

Les méthodes de résolution des clés de contexte, de classification des composants utilisables sur le réseau et de recherche dans les registres étaient notamment absentes de la couverture initiale.

`usesCompactFormat` a été ciblée parce qu'elle est publique, statique, sans dépendance externe et qu'elle appelle `findContextKeyInterface`, dont des branches portaient des mutants non couverts par les tests originaux.

### 1.2 `ComponentRegistry`

La suite originale `ComponentRegistryTest` contenait six tests. JaCoCo mesurait initialement 69 % des instructions et 61 % des branches de la classe. PIT a également révélé des branches non testées et des oracles insuffisamment précis.

| Mesure initiale | Valeur |
|---|---:|
| Couverture de lignes PIT | 63/83 (76 %) |
| Mutants générés | 27 |
| Mutants tués | 19/27 (70 %) |
| Mutants survivants | 3 |
| Mutants sans couverture | 5 |
| Test strength | 19/22 (86 %) |

Les lacunes concernaient entre autres un nom de composant absent, la recherche inverse d'un nom convivial, les suffixes `:default` et `:key=...`, les composants par défaut et le numéro de ligne d'un index invalide.

`getComponentClass`, `getComponentInfo` et `hasComponent` ont initialement été sélectionnées comme points d'entrée publics simples du registre pour les essais avec ChatUniTest. Comme ces essais n'ont produit aucun test compilable, le rapport PIT a ensuite orienté les tests manuels vers les comportements réellement insuffisamment testés, notamment `hasComponent`, `getFriendlyName`, `getDefaultComponents` et le traitement des suffixes de l'index. Les lignes 119 et 123 de `getDefaultComponents` étaient notamment sans couverture dans le rapport initial.

## 2. Environnement

| Élément | Environnement principal | Expérience `ComponentRegistry` |
|---|---|---|
| Système | Windows 11, PowerShell | Windows 11, PowerShell |
| Java | Temurin 25.0.2 | Temurin 25.0.4.1 |
| Maven | Wrapper 3.9.12 | Maven 3.9.16 |
| Tests | JUnit Jupiter | JUnit Jupiter |
| Génération | ChatUniTest 2.1.1, Ollama, `code-llama` | ChatUniTest 2.1.1, `llama3` exposé sous l'alias `code-llama` |
| Mutation | PIT 1.25.8, plug-in JUnit 5 1.2.3 | PIT 1.30.0, plug-in JUnit 5 1.2.3 |

Le résultat final de `ComponentRegistry` obtenu avec PIT 1.30.0 a été reproduit dans le dépôt commun avec PIT 1.25.8 : 27 mutants tués sur 27 dans les deux cas.

Dans l'environnement principal, `code-llama` est le nom de modèle compatible présenté à ChatUniTest par le serveur Ollama local.

## 3. Génération ChatUniTest pour `ComponentNameResolver`

### 3.1 Déroulement exact

La première génération visant l'ensemble de `ComponentNameResolver` a échoué à cause d'un délai d'attente de la connexion avec le modèle local.

Une seconde génération a donc été limitée à la méthode `usesCompactFormat`, avec les choix suivants :

- un seul test demandé par génération (`testNumber=1`);
- exécution parallèle désactivée (`thread=false`);
- deux rondes maximales configurées (`maxRounds=2`) : une génération initiale et une ronde automatique de réparation.

Cette seconde commande de génération s'est terminée, contrairement à la tentative visant toute la classe. Cependant, « génération terminée » ne signifie pas « test exécutable » : les tests Java produits ne compilaient toujours pas.

ChatUniTest a d'abord produit un test pour `usesCompactFormat`. La compilation a échoué. Les erreurs ont alors été transmises automatiquement au modèle local et une ronde de réparation a été effectuée. Le test réparé ne compilait toujours pas. L'opération n'a donc produit aucun test directement exécutable sans intervention manuelle. Les deux rondes ont pris environ **5 minutes et 29 secondes**.

Les erreurs observées comprenaient notamment :

- des imports Mockito alors que Mockito n'est pas une dépendance du module;
- une tentative d'appeler le constructeur privé de `ComponentNameResolver`;
- l'accès au champ privé `REGISTRIES`;
- l'utilisation de réflexion alors que `usesCompactFormat` est publique et statique;
- des méthodes ou constructeurs inexistants pour `ComponentInfo`.

Les sorties brutes et les erreurs de compilation ont été conservées dans les journaux de l'expérience et dans le répertoire `chatunitest-info` généré par l'outil.

### 3.2 Répartition des neuf tests finaux

ChatUniTest a produit un candidat initial visant `usesCompactFormat`, puis une version réparée de ce candidat. Aucun des deux ne compilait. Deux scénarios fonctionnels, un positif et un négatif, ont ensuite été extraits de ces propositions et adaptés manuellement. Les traces conservées ne permettent pas d'attribuer séparément chaque scénario à une ronde précise. Sept tests supplémentaires ont ensuite été écrits manuellement à partir du contrat de la classe et des rapports PIT.

| Origine | Nombre | Tests |
|---|---:|---|
| Proposés par ChatUniTest, puis corrigés manuellement | 2 | `parserUsesCompactFormat`, `unrelatedClassDoesNotUseCompactFormat` |
| Ajoutés manuellement | 7 | `nullTypeIsRejected` et six scénarios guidés par PIT |
| **Total final** | **9** | Tous dans `ComponentNameResolverChatUniTestTest` |

Les sept tests manuels ont été regroupés avec les deux scénarios adaptés dans la même classe afin de centraliser les tests supplémentaires de `ComponentNameResolver` et de simplifier leur sélection par Maven et PIT.

## 4. Critique des deux tests proposés par ChatUniTest

### 4.1 `parserUsesCompactFormat`

- **Intention :** vérifier qu'une interface reconnue comme clé de contexte utilise le format compact.
- **Donnée choisie :** `Parser.class`, car `Parser` fait partie des interfaces déclarées dans `CONTEXT_KEY_INTERFACES`.
- **Oracle :** `assertTrue(ComponentNameResolver.usesCompactFormat(Parser.class))`.
- **Critique :** l'idée fonctionnelle était pertinente, mais le test brut utilisait un mock de `Class`, une instance impossible de `ComponentNameResolver` et de la réflexion inutile. Le test final appelle directement l'API publique avec un type réel.

### 4.2 `unrelatedClassDoesNotUseCompactFormat`

- **Intention :** vérifier qu'une classe sans rapport avec les interfaces de contexte n'utilise pas le format compact.
- **Donnée choisie :** `String.class`, type stable du JDK qui n'implémente aucune interface de contexte Tika.
- **Oracle :** `assertFalse(ComponentNameResolver.usesCompactFormat(String.class))`.
- **Critique :** le scénario positif/négatif proposé était utile, mais son montage avec Mockito ne représentait pas le comportement réel de `Class.isAssignableFrom`. L'utilisation d'un vrai type rend l'oracle plus fiable et plus lisible.

### 4.3 Mutants détectés grâce aux deux tests corrigés

La suite originale tuait un mutant. Après ajout des deux tests corrigés, PIT indiquait cinq mutants tués au total. Les deux tests issus de ChatUniTest ont donc tué **quatre mutants supplémentaires**.

Les numéros de ligne ci-dessous correspondent au rapport PIT produit pendant l'expérience et peuvent changer si le fichier source est modifié.

| Méthode mutée | Mutation | Test qui la tue | Pourquoi l'oracle échoue sur le mutant |
|---|---|---|---|
| `findContextKeyInterface` (ligne 335) | condition inversée | `unrelatedClassDoesNotUseCompactFormat` | `String.class` serait incorrectement reconnu comme interface de contexte et le résultat deviendrait `true`. |
| `findContextKeyInterface` (ligne 336) | valeur de retour remplacée par `null` | `parserUsesCompactFormat` | `Parser.class` ne serait plus reconnu et `usesCompactFormat` retournerait `false`. |
| `usesCompactFormat` (ligne 350) | condition inversée | `parserUsesCompactFormat` | le cas positif `Parser.class` retournerait `false`. |
| `usesCompactFormat` (ligne 350) | retour booléen remplacé par `true` | `unrelatedClassDoesNotUseCompactFormat` | le cas négatif `String.class` retournerait incorrectement `true`. |

Après ces deux tests, les résultats étaient :

| Mesure | Suite originale | Après les deux tests corrigés |
|---|---:|---:|
| Couverture de lignes PIT | 43/119 (36 %) | 49/119 (41 %) |
| Mutants tués | 1/39 (3 %) | 5/39 (13 %) |
| Mutants sans couverture | 37 | 33 |
| Test strength | 50 % | 83 % |

Les deux tests sont donc pertinents, mais leur portée reste limitée à `usesCompactFormat` et à la méthode auxiliaire qu'elle appelle.

Le `test strength` de 83 % ne contredit pas le score de mutation de 13 %. Il ne considère que les mutants couverts par au moins un test : cinq des six mutants couverts ont été tués. Le score de mutation considère les 39 mutants générés, y compris les 33 mutants qui n'étaient couverts par aucun test.

## 5. Sept tests manuels pour `ComponentNameResolver`

### 5.1 `nullTypeIsRejected`

- **Intention :** documenter le comportement lorsque le type fourni est nul.
- **Donnée choisie :** `null`, seule valeur permettant d'exercer ce cas limite.
- **Oracle :** `assertThrows(NullPointerException.class, ...)`.
- **Justification :** l'appel interne à `Class.isAssignableFrom` exige une classe non nulle; le test empêche qu'une modification transforme silencieusement ce cas en résultat booléen.

### 5.2 `unregisteredMessageMatchesRegistryState`

- **Intention :** distinguer la branche où aucun registre n'est disponible de celle où des composants connus peuvent être affichés.
- **Données choisies :** le composant de test connu `configurable-test-parser` et un nom volontairement impossible, `unknown-component-for-mutation-test`.
- **Oracle :** vérification de fragments propres à l'état réel du registre, notamment `registered component(s):`, le nom connu ou le message `No components are currently registered`.
- **Justification :** un simple `assertThrows` n'aurait pas distingué les deux branches du message.

### 5.3 `determineContextKeyUsesDocumentedResolutionOrder`

- **Intention :** vérifier les trois niveaux de priorité de la résolution d'une clé de contexte.
- **Données choisies :**
  - un `ComponentInfo` avec `MetadataFilter.class` comme clé explicite;
  - `ParserDecorator.class`, qui permet la détection automatique de `Parser.class`;
  - `String.class`, qui force le repli sur la classe elle-même.
- **Oracle :** trois `assertEquals` sur les classes exactes attendues.
- **Justification :** chaque donnée active une branche différente et vérifie également l'ordre de priorité.

### 5.4 `wireContextKeysRespectSecurityClassification`

- **Intention :** vérifier la politique de sécurité appliquée aux clés de contexte reçues depuis le réseau.
- **Données choisies :** `MetadataFilter.class` comme clé autorisée, `Parser.class` comme clé bloquée et `String.class` comme type ordinaire.
- **Oracle :** assertions `true` et `false` sur `isWireInstantiable` et `isWireBlocked`.
- **Justification :** les trois catégories empêchent un mutant de satisfaire le test avec un retour booléen constant.

### 5.5 `contextKeySetsExposeExpectedEntries`

- **Intention :** vérifier que les ensembles publics reflètent les classifications internes.
- **Données choisies :** `Parser.class` et `MetadataFilter.class`, représentants respectifs d'une clé bloquée et d'une clé autorisée.
- **Oracle :** vérification de leur présence dans les trois ensembles publics pertinents.
- **Justification :** les valeurs choisies relient l'API d'inspection aux décisions de sécurité testées précédemment.

### 5.6 `registeredComponentsCanBeQueried`

- **Intention :** tester les opérations de recherche avec un véritable `ComponentRegistry`.
- **Données choisies :** l'index de test `parsers`, `configurable-test-parser` comme nom existant, un nom inexistant et `ConfigurableTestParser.class`.
- **Oracle :** présence/absence avec `hasComponent`, `Optional` présent ou vide avec `getComponentInfo`, résultats de `hasImplementationsOf` et classe exacte retournée par `getContextKey`.
- **Justification :** un registre réel évite les mocks et couvre plusieurs méthodes publiques avec des données générées par l'annotation processor du projet.

### 5.7 `registeredComponentMessageUsesCorrectSeparators`

- **Intention :** vérifier le format de la liste de composants dans le message d'erreur.
- **Données choisies :** un registre réel contenant plusieurs parseurs et un nom inconnu.
- **Oracle :** la liste ne commence pas par une virgule, contient `, ` entre les éléments et contient `configurable-test-parser`.
- **Justification :** ces assertions tuent les mutations sur la condition et l'incrément qui contrôlent l'ajout des séparateurs.

### 5.8 Résultat final pour `ComponentNameResolver`

| Mesure finale | Valeur |
|---|---:|
| Couverture de lignes PIT | 111/119 (93 %) |
| Mutants générés | 39 |
| Mutants tués | 39/39 (100 %) |
| Mutants survivants | 0 |
| Mutants sans couverture | 0 |
| Test strength | 100 % |

Les sept tests manuels étendent la couverture aux méthodes et branches signalées par PIT. La suite complète tue les 34 mutants qui n'étaient pas encore tués après l'ajout des deux tests corrigés.

## 6. Génération ChatUniTest pour `ComponentRegistry`

Le modèle `llama3` a été exposé à ChatUniTest sous l'alias accepté `code-llama`.

- Pour `getComponentClass` et `getComponentInfo`, cinq essais avec plusieurs tours de correction n'ont produit aucun test compilable.
- L'exécution visant `hasComponent` a été interrompue après des délais et erreurs répétés.
- `getAllComponents` et `getDefaultComponents` ont été ignorées par le plug-in.
- Les journaux contiennent notamment des réponses sans code Java extractible et des `SocketTimeoutException`.

Aucun test ChatUniTest exécutable n'a donc été obtenu pour cette classe et aucun score PIT ne peut honnêtement être attribué à la sortie de l'IA. Ce résultat négatif fait partie de l'expérience.

## 7. Quatre tests manuels pour `ComponentRegistry`

Les tests créent un fichier `META-INF/tika/mutation-test-index.idx` dans un dossier temporaire fourni par `@TempDir`, puis le chargent avec un `URLClassLoader` fermé automatiquement. Les classes utilisées comme composants proviennent du JDK afin de rendre les données déterministes.

### 7.1 `testHasComponentReturnsFalseForUnknownName`

- **Intention :** vérifier les cas présent et absent de `hasComponent`.
- **Données choisies :** `known=java.lang.String` et le nom `does-not-exist`.
- **Oracle :** `assertTrue` pour `known` et `assertFalse` pour le nom absent.
- **Mutant ciblé :** remplacement du retour de `hasComponent` par `true`.

### 7.2 `testGetFriendlyName`

- **Intention :** vérifier la recherche inverse d'un nom convivial.
- **Données choisies :** deux associations différentes pour `String.class` et `ArrayList.class`, puis `Integer.class` comme classe absente.
- **Oracle :** noms exacts `string-comp` et `list-comp`, puis `null` pour `Integer.class`.
- **Mutant ciblé :** remplacement d'un résultat par une chaîne vide ainsi que les erreurs de recherche inverse.

### 7.3 `testDefaultAndKeySuffixes`

- **Intention :** vérifier le traitement des suffixes `:default` et `:key=...`, ainsi que l'immuabilité des résultats exposés.
- **Données choisies :** quatre lignes couvrant l'absence de suffixe, `:default`, `:key=java.lang.Number` et leur combinaison.
- **Oracle :** ensemble exact des valeurs par défaut `{a, d}`, taille totale de quatre, indicateurs `isDefault`, classes et clés de contexte exactes, puis `UnsupportedOperationException` lors d'une modification de la map.
- **Mutants ciblés :** branches sur `default`, résolution de `contextKey`, retour de `getDefaultComponents` et enveloppe non modifiable.

### 7.4 `testInvalidSuffixReportsLineNumber`

- **Intention :** vérifier le refus d'un suffixe inconnu et l'exactitude du numéro de ligne signalé.
- **Données choisies :** un commentaire, une ligne valide, puis une troisième ligne contenant `:bogus`.
- **Oracle :** `TikaConfigException` dont le message contient `line 3: unknown suffix 'bogus'`.
- **Mutants ciblés :** inversion de la condition du suffixe inconnu et remplacement de `lineNumber++` par `lineNumber--`.

### 7.5 Mutants nouvellement détectés

| Ligne PIT | Mutation initialement non détectée | Test qui la détecte | Raison |
|---:|---|---|---|
| 133 | `hasComponent` renvoie toujours `true` | `testHasComponentReturnsFalseForUnknownName` | le nom absent doit produire `false`. |
| 144 | `getFriendlyName` renvoie une chaîne vide | `testGetFriendlyName` | les noms exacts et le cas `null` sont vérifiés. |
| 180 | `lineNumber++` devient `lineNumber--` | `testInvalidSuffixReportsLineNumber` | le message doit indiquer exactement la ligne 3. |
| 237 | condition de la clé de contexte inversée | `testDefaultAndKeySuffixes` | les clés `Number.class` sont comparées exactement. |
| 119, 123 | `getDefaultComponents` sans couverture | `testDefaultAndKeySuffixes` | l'ensemble exact `{a, d}` et son immuabilité sont vérifiés. |
| 220 | condition `:default` inversée | `testDefaultAndKeySuffixes` | chaque indicateur `isDefault` est vérifié. |
| 222 | condition du suffixe inconnu inversée | `testInvalidSuffixReportsLineNumber` | le suffixe `:bogus` doit provoquer l'exception attendue. |

### 7.6 Résultat final pour `ComponentRegistry`

| Mesure | Tests originaux | Tests originaux et quatre tests manuels |
|---|---:|---:|
| Couverture de lignes PIT | 63/83 (76 %) | 74/83 (89 %) |
| Mutants tués | 19/27 (70 %) | 27/27 (100 %) |
| Mutants survivants | 3 | 0 |
| Mutants sans couverture | 5 | 0 |
| Test strength | 86 % | 100 % |

## 8. Comparaison qualitative des oracles

| Critère | Sorties ChatUniTest | Tests manuels |
|---|---|---|
| Compilation | Échecs fréquents; réparation automatique insuffisante | Compilation après formatage et intégration |
| Connaissance de l'API | Mocks de `Class`, réflexion, constructeur et membres privés parfois inventés | API publiques et vrais mécanismes de chargement du projet |
| Précision | Deux idées pertinentes pour les sorties `true` et `false` de `usesCompactFormat` | Classes exactes, ensembles exacts, messages et exceptions précis |
| Étendue | Une méthode ciblée pour `ComponentNameResolver`; aucun test exécutable pour `ComponentRegistry` | Branches et méthodes choisies à partir des mutants survivants ou sans couverture |
| Résultat PIT | Quatre mutants supplémentaires tués par les deux tests corrigés | Tous les mutants restants tués dans les deux classes |

Les idées de ChatUniTest étaient utiles pour amorcer deux scénarios simples, mais elles n'étaient pas directement utilisables. Les tests manuels ont produit les gains principaux parce que leurs données et leurs oracles ont été choisis en fonction de changements fautifs précis observés dans les rapports PIT.

Un score de mutation de 100 % ne prouve pas que le logiciel est exempt de bogues. Il indique seulement que la suite détecte tous les mutants générés par les opérateurs utilisés pendant cette exécution.

## 9. Exécution et reproduction

Les commandes suivantes sont lancées depuis la racine du dépôt.

### 9.1 Suite complète du module

```powershell
.\mvnw.cmd -pl tika-serialization test
```

Dernier résultat local conservé :

```text
Tests run: 229, Failures: 0, Errors: 0, Skipped: 4
BUILD SUCCESS
```

### 9.2 Tests ajoutés pour `ComponentNameResolver`

```powershell
.\mvnw.cmd -pl tika-serialization test `
  "-Dtest=ComponentNameResolverChatUniTestTest" `
  "-Dsurefire.failIfNoSpecifiedTests=false"
```

Résultat : neuf tests exécutés, aucun échec et aucune erreur.

### 9.3 PIT avec seulement les tests originaux de `ComponentNameResolver`

```powershell
.\mvnw.cmd -pl tika-serialization test-compile `
  org.pitest:pitest-maven:mutationCoverage `
  "-DtargetClasses=org.apache.tika.serialization.ComponentNameResolver" `
  "-DtargetTests=org.apache.tika.serialization.ComponentNameResolverTest" `
  "-Djacoco.skip=true"
```

Résultat : 39 mutants générés et 1 mutant tué (3 %).

Le score intermédiaire de 5/39 a été mesuré dans un état de travail temporaire où seuls les deux scénarios adaptés de ChatUniTest étaient actifs. Cet état n'a pas fait l'objet d'un commit séparé. Le commit `bd9d083d5` contient déjà les neuf tests de `ComponentNameResolverChatUniTestTest` et ne permet donc pas, à lui seul, de reproduire ce score intermédiaire. Cette mesure provient du résumé PIT conservé pendant l'expérience, mais ce rapport intermédiaire n'est pas versionné dans le dépôt. Cette limite de reproductibilité n'affecte pas les résultats initial et final, qui peuvent être reproduits avec les commandes 9.3 et 9.4.

### 9.4 PIT final de `ComponentNameResolver`

```powershell
.\mvnw.cmd -pl tika-serialization test-compile `
  org.pitest:pitest-maven:mutationCoverage `
  "-DtargetClasses=org.apache.tika.serialization.ComponentNameResolver" `
  "-DtargetTests=org.apache.tika.serialization.ComponentNameResolver*Test" `
  "-Djacoco.skip=true"
```

Résultat : 39 mutants générés et 39 mutants tués.

### 9.5 Tests ajoutés pour `ComponentRegistry`

```powershell
.\mvnw.cmd -pl tika-serialization test `
  "-Dtest=ComponentRegistryMutationTest" `
  "-Dsurefire.failIfNoSpecifiedTests=false"
```

Résultat : quatre tests exécutés, aucun échec et aucune erreur.

### 9.6 PIT avec seulement les tests originaux de `ComponentRegistry`

```powershell
.\mvnw.cmd -pl tika-serialization test-compile `
  org.pitest:pitest-maven:mutationCoverage `
  "-DtargetClasses=org.apache.tika.config.loader.ComponentRegistry" `
  "-DtargetTests=org.apache.tika.config.loader.ComponentRegistryTest" `
  "-Djacoco.skip=true"
```

Résultat : 27 mutants générés et 19 mutants tués (70 %).

### 9.7 PIT final de `ComponentRegistry`

```powershell
.\mvnw.cmd -pl tika-serialization test-compile `
  org.pitest:pitest-maven:mutationCoverage `
  "-DtargetClasses=org.apache.tika.config.loader.ComponentRegistry" `
  "-DtargetTests=org.apache.tika.config.loader.ComponentRegistry*Test" `
  "-Djacoco.skip=true"
```

Résultat : 27 mutants générés et 27 mutants tués.

## 10. Intégration Git et validation continue

Les changements principaux sont identifiés par les commits suivants :

| Commit | Contenu |
|---|---|
| `bd9d083d5` | premiers tests inspirés de ChatUniTest et configuration PIT |
| `a02cc4a44` | finalisation des tests de mutation de `ComponentNameResolver` |
| `8c2821617` | ajout des tests de mutation de `ComponentRegistry` |
| `1aff5f532` | ajout du workflow GitHub Actions dédié aux treize nouveaux tests |

Les tests s'exécutent localement avec Maven, Checkstyle et Spotless. Un workflow dédié,
[`IFT3913 Task 2 Tests`](https://github.com/ReddKingK17/tika/actions/runs/37874508476),
exécute explicitement les deux nouvelles classes de tests avec Java 17 à chaque `push` et
à chaque `pull_request` sur `main`.

Le workflow se trouve dans [`.github/workflows/tache2-tests.yml`](.github/workflows/tache2-tests.yml). Java 17 est utilisé en intégration continue parce qu'il s'agit de la version de base du projet Apache Tika, tandis que les expériences locales décrites à la section 2 ont été réalisées avec Java 25.

L'exécution associée au commit `1aff5f532` s'est terminée avec succès :

- `ComponentNameResolverChatUniTestTest` : 9 tests réussis;
- `ComponentRegistryMutationTest` : 4 tests réussis;
- total : 13 tests, 0 échec, 0 erreur et 0 test ignoré.

## 11. Difficultés rencontrées

- le JAR du module devait être installé avant l'exécution de ChatUniTest;
- le nom de modèle accepté par le plug-in devait être compatible avec sa liste de modèles;
- la génération complète de `ComponentNameResolver` a dépassé le délai d'attente;
- les tests proposés utilisaient Mockito, des membres privés et des API inexistantes;
- les sorties visant `ComponentRegistry` ne contenaient pas de test compilable;
- les fins de ligne CRLF ont d'abord été refusées par Checkstyle;
- les analyses ont été reproduites avec deux versions de PIT afin de confirmer la stabilité du résultat.

## 12. Déclaration d'utilisation de l'intelligence artificielle

ChatUniTest 2.1.1 a été utilisé avec des modèles ouverts exécutés localement par Ollama afin de produire des propositions de tests. Les propositions ont été compilées, critiquées et corrigées manuellement; aucun test produit par ChatUniTest n'a été accepté sans vérification.

ChatGPT a également été utilisé comme assistance pour :

- interpréter les erreurs Maven, ChatUniTest, Checkstyle et PIT;
- diagnostiquer les échecs de compilation des propositions ChatUniTest;
- structurer et réviser la documentation.



## 13. Résumé des résultats

| Classe | PIT initial | Après les tests ChatUniTest corrigés manuellement | PIT final | Couverture de lignes PIT finale |
|---|---:|---:|---:|---:|
| `ComponentNameResolver` | 1/39 (3 %) | 5/39 (13 %) au total | 39/39 (100 %) | 111/119 (93 %) |
| `ComponentRegistry` | 19/27 (70 %) | Non mesurable | 27/27 (100 %) | 74/83 (89 %) |

Au total, la suite intégrée exécute 229 tests, sans échec ni erreur. Les deux classes atteignent un score de mutation et un `test strength` de 100 % pour les mutants générés lors des expériences.
