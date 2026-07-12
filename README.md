# Codex subagents configuration

Configuration personnelle des sous-agents Codex et de leur routage.

Ce dépôt constitue une source de vérité lisible et versionnée pour mon workflow
agentique local. Il ne contient ni credentials, ni données de projet, ni copie
complète de ma configuration machine.

## Philosophie du workflow

Le main agent reste le coordinateur. Il conserve le contexte global, comprend
la demande utilisateur, choisit les sous-agents pertinents et assemble leurs
résultats. Il est recommandé de le faire fonctionner avec `gpt-5.6-sol` et un
niveau de raisonnement `low` : son rôle est principalement de router, suivre
l'avancement et synthétiser les rapports, plutôt que de refaire lui-même toute
l'exploration ou l'implémentation. Il n'est donc pas nécessaire de lui attribuer
un raisonnement élevé par défaut : les décisions qui exigent réellement plus
de profondeur doivent être déléguées ou faire l'objet d'une escalade ciblée.

L'objectif principal est d'optimiser la consommation de tokens et de quota sans
sacrifier la qualité. On réserve le raisonnement coûteux aux problèmes qui le
justifient par des éléments observables : ambiguïté architecturale, couplage
inattendu, risque de sécurité, migration, concurrence, régression difficile à
expliquer ou échec de validation dont la cause reste incertaine.

Les rôles spécialisés utilisent le compromis suivant entre coût, rapidité et
profondeur de raisonnement :

| Rôle | Modèle recommandé | Raisonnement | Responsabilité |
| --- | --- | --- | --- |
| Main agent | `gpt-5.6-sol` | `low` | Coordination, routage et synthèse |
| `code-explorer` | `gpt-5.6-luna` | `high` | Exploration large et traçage des contrats |
| `implementer` | `gpt-5.6-luna` | `high` | Fonctionnalités, corrections et tests |
| `quick-implementer` | `gpt-5.6-luna` | `high` | Petits changements mécaniques ciblés |
| `code-reviewer` | `gpt-5.6-sol` | `low` | Revue indépendante en lecture seule |
| `commit-pusher` | `gpt-5.6-luna` | `low` | Commit et push explicitement demandés |

Pour les tâches d'implémentation bornées, `gpt-5.6-luna` avec un raisonnement
`high` est considéré comme suffisant dans ce workflow, y compris pour les
corrections qui demandent une analyse locale approfondie. On n'utilise pas un
palier plus coûteux comme `terra` par défaut : il ne doit être envisagé que si
des évaluations propres au dépôt montrent un meilleur taux de réussite, une
meilleure latence ou un coût total inférieur à la combinaison Luna/Sol adaptée.

De même, un raisonnement `high` pour Sol est une escalade, pas le réglage
normal du main agent. Avant d'augmenter le raisonnement, il faut vérifier que
le problème n'est pas simplement dû à un manque de contexte, à des critères
d'acceptation vagues, à une permission absente ou à une erreur d'environnement.

### Optimiser le coût de la tâche terminée

Le coût par token n'est pas le seul indicateur pertinent. Un modèle moins
coûteux qui nécessite plusieurs tentatives, corrections ou revues peut coûter
plus cher pour obtenir un résultat fiable qu'un modèle plus puissant qui réussit
en une seule passe.

Le choix d'un modèle doit donc prendre en compte :

- la consommation de tokens et de quota ;
- la probabilité d'une première implémentation correcte ;
- le nombre de cycles de correction ;
- le coût des tests et de la revue ;
- le temps nécessaire pour obtenir un résultat vérifiable.

### Escalader selon le périmètre de risque

La difficulté apparente n'est pas le seul critère de routage. Une refactorisation
complexe mais isolée peut rester adaptée à Luna, tandis qu'une modification
apparemment simple touchant l'authentification, les paiements, les permissions,
une migration ou une configuration de production peut justifier Sol.

L'escalade doit refléter l'impact potentiel d'une erreur, et pas seulement le
nombre de fichiers concernés.

### Escalader uniquement sur la base d'éléments observables

Lorsqu'un agent rencontre une difficulté, il faut d'abord vérifier si elle vient
d'un contexte incomplet, de critères d'acceptation vagues, de permissions
manquantes ou d'un problème d'environnement. Ces problèmes doivent être
corrigés directement plutôt que compensés par davantage de raisonnement.

Une escalade est justifiée par exemple par :

- un choix architectural encore non résolu ;
- un couplage inattendu entre plusieurs composants ;
- plusieurs hypothèses incorrectes malgré de nouvelles informations ;
- un échec de validation dont la cause reste inexpliquée ;
- un risque important de sécurité, compatibilité, migration ou concurrence.

Ces valeurs sont des recommandations de configuration, pas des garanties du
runtime. Elles doivent être relues lorsque les modèles disponibles ou les
conventions de Codex évoluent.

Les sous-agents ont des responsabilités spécialisées :

- `code-explorer` : exploration large, recherche de fichiers, traçage de
  contrats et compréhension d'un dépôt ;
- `quick-implementer` : changements mécaniques, bien spécifiés et limités ;
- `implementer` : fonctionnalités, corrections et changements nécessitant des
  tests ou plusieurs fichiers ;
- `code-reviewer` : revue indépendante des changements terminés ;
- `commit-pusher` : commit et push, uniquement après demande explicite de
  l'utilisateur.

Le flux par défaut est :

```text
demande utilisateur
        ↓
analyse et routage par le main agent
        ↓
exploration si nécessaire
        ↓
implémentation
        ↓
tests et validation
        ↓
revue indépendante si le changement est significatif
        ↓
corrections et tests de non-régression
```

La délégation automatique est une règle de comportement, pas une garantie
mécanique. Le main peut traiter directement une tâche triviale ou déléguer une
tâche qui nécessite une exploration ou une implémentation spécialisée.

## Contenu du dépôt

- `agents/` contient les définitions TOML des sous-agents ;
- `rules/SUBAGENT_ROUTING.md` contient les règles de sélection et de séquencement ;
- `AGENTS.md` contient les instructions globales personnelles et la référence
  vers le routage installé ;
- ce README documente les manipulations manuelles et les choix du workflow.

Le dépôt ne versionne volontairement pas :

- `~/.codex/config.toml` en entier ;
- les données ou configurations spécifiques à un projet ;
- les chemins et réglages qui ne seraient utiles que sur une autre machine.

## Installation manuelle

Les opérations suivantes supposent que `$CODEX_HOME` vaut `~/.codex`. Si une
autre valeur est utilisée, remplacer ce chemin dans toutes les commandes.

### 1. Sauvegarder la configuration existante

```sh
cp -R ~/.codex ~/.codex.backup-$(date +%Y%m%d-%H%M%S)
```

### 2. Installer les agents

```sh
mkdir -p ~/.codex/agents
cp agents/*.toml ~/.codex/agents/
```

Avant de remplacer un agent existant, vérifier son contenu et conserver toute
personnalisation locale utile.

### 3. Installer les règles de routage

```sh
mkdir -p ~/.codex/rules
cp rules/SUBAGENT_ROUTING.md ~/.codex/rules/SUBAGENT_ROUTING.md
```

### 4. Référencer le routage dans `AGENTS.md`

Ajouter le bloc suivant dans `~/.codex/AGENTS.md`, en adaptant le chemin si
nécessaire :

```md
# BEGIN subagents_configs
@{Path}/.codex/rules/SUBAGENT_ROUTING.md
# END subagents_configs
```

Le fichier global peut également contenir les préférences personnelles de
rapport final présentes dans ce dépôt.

### 5. Activer la configuration multi-agent

La version actuelle de Codex active normalement le multi-agent par défaut.
Pour reproduire explicitement le réglage utilisé avec cette configuration,
ajouter à `~/.codex/config.toml` :

```toml
[features.multi_agent_v2]
hide_spawn_agent_metadata = false
tool_namespace = "agents"
```

Ne pas remplacer le `config.toml` existant : ajouter uniquement ce bloc si la
table n'existe pas déjà.

### 6. Redémarrer et vérifier

Redémarrer Codex ou ouvrir une nouvelle conversation. Tester ensuite une tâche
qui nécessite une exploration large, puis une implémentation et une revue.

Exemple de test reproductible :

```text
Crée un nouveau projet React nommé `pokedex-app` dans le répertoire courant.

Construis une interface web moderne de Pokédex avec :

- une grille de cartes Pokémon ;
- une recherche par nom ;
- des filtres par type ;
- une fiche détaillée au clic ;
- une pagination ou un chargement progressif ;
- des états de chargement, erreur et liste vide ;
- un design responsive desktop/mobile ;
- des composants React bien séparés ;
- des données provenant de l'API publique PokeAPI ;
- des tests pour la recherche, les filtres et l'affichage d'une carte ;
- des tests déterministes avec les appels réseau mockés ;
- un README avec les commandes d'installation, de lancement et de test.

Utilise Vite, TypeScript et une solution CSS simple.

Ne fais aucun commit ni push.
```

Le main agent devrait sélectionner les agents spécialisés lorsque la tâche le
justifie. Une demande explicite reste possible lorsqu'un agent précis est
nécessaire.

## Sécurité et gouvernance

- Les agents ne doivent pas recevoir plus de permissions que nécessaire.
- Les agents d'exploration et de revue doivent rester en lecture seule lorsque
  le contexte le permet.
- `commit-pusher` ne doit jamais être lancé pour une simple demande de codage.
- Aucun commit ou push ne doit être effectué sans demande explicite.
- Les tests et les builds doivent être exécutés avant de considérer une tâche
  terminée.
- Une revue indépendante est particulièrement utile après une modification
  multi-fichiers, une modification d'architecture ou un changement risqué.
- Les consignes globales doivent rester courtes ; les règles propres à un dépôt
  doivent être placées dans son propre `AGENTS.md`.

## Rapport final attendu

À la fin de chaque tâche de développement, le main agent doit résumer :

- les fichiers créés ou modifiés ;
- les commandes et tests exécutés ;
- les éventuels risques, limitations ou points restant à améliorer.

## Évolution

Les définitions d'agents et les règles de routage doivent être modifiées,
testées sur un petit dépôt, puis versionnées. Les changements de modèle ou de
fonctionnalité Codex doivent être vérifiés avec la documentation actuelle avant
d'être propagés à l'ensemble de la configuration.
