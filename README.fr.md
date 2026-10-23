# Sylius Pack : IA et headless

> English: [README.md](README.md) <!-- vitrine:lang -->

[![Build](https://github.com/thibmonier/sylius-pack-ai-headless/actions/workflows/build.yaml/badge.svg)](https://github.com/thibmonier/sylius-pack-ai-headless/actions/workflows/build.yaml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

**Quatre plugins Sylius 2 pour les assistants d'IA et les intégrations, installés avec un seul `composer require` : flux produits, copilote d'IA pour le catalogue, outils B2B pour le serveur MCP de Sylius et webhooks sortants fiables.**

Ce paquet est un métapaquet Composer : il ne contient aucun code. Il requiert les quatre plugins ci-dessous, avec des bornes de versions qui se résolvent ensemble, pour ne pas les choisir un par un. Il est gratuit et open source.

## Ce que fait un métapaquet, et ce qu'il ne fait pas

- Il fait : une commande installe les quatre plugins à des versions dont la résolution ensemble est vérifiée.
- Il ne fait pas : déclarer les bundles, importer la configuration et les routes, lancer les migrations ni planifier les tâches. Ces étapes restent manuelles, plugin par plugin, et sont listées dans [Ce qui reste à faire à la main](#ce-qui-reste-à-faire-à-la-main). Tant que la recette Flex du pack n'est pas publiée sur `symfony/recipes-contrib` (voir [Recette Flex](#recette-flex)), `composer require` ne les fait pas.
- Il ne garantit pas votre boutique : l'intégration continue installe le pack dans un projet Sylius vierge, avec les gestes ci-dessous et rien d'autre (voir [Vérifié en CI](#vérifié-en-ci)) ; votre application peut différer.

## Prérequis

- PHP 8.2 ou plus
- Sylius 2.2.9 ou plus (2.3 pris en charge)
- `symfony/ai-bundle` et `symfony/ai-platform` en `^0.14` (exigés par le copilote d'IA) : le pack ne s'installe pas à côté d'un paquet qui impose une autre version mineure de Symfony AI.

## Installation

```bash
composer require tmonier/sylius-pack-ai-headless
```

Suivre ensuite [Ce qui reste à faire à la main](#ce-qui-reste-à-faire-à-la-main).

## Plugins du pack

| Plugin | Borne | Rôle |
|--------|-------|------|
| [sylius-agentic-feeds-plugin](https://github.com/thibmonier/sylius-agentic-feeds-plugin) | `^1.0` | Flux produits (Google Merchant Center, OpenAI, Stripe) et JSON-LD des produits |
| [sylius-ai-catalog-copilot-plugin](https://github.com/thibmonier/sylius-ai-catalog-copilot-plugin) | `^1.0` | Suggestions relues de descriptions, de méta et de traductions via Symfony AI |
| [sylius-b2b-mcp-plugin](https://github.com/thibmonier/sylius-b2b-mcp-plugin) | `^1.2` | Outils B2B pour le serveur MCP officiel de Sylius (OAuth 2.1 avec PKCE) |
| [sylius-webhook-outbox-plugin](https://github.com/thibmonier/sylius-webhook-outbox-plugin) | `^1.0` | Webhooks sortants avec outbox transactionnelle, signatures, nouvelles tentatives et rejeu |

Suggéré, non installé (`suggest`) :

| Plugin | Pourquoi |
|--------|----------|
| [sylius-audit-log-plugin](https://github.com/thibmonier/sylius-audit-log-plugin) | Le copilote y consigne la modification du produit faite à l'acceptation d'une suggestion, et les appels d'outils MCP y sont reflétés. Il fait partie du [pack Conformité et outillage](https://github.com/thibmonier/sylius-pack-compliance-tooling). |

## Ce qui reste à faire à la main

Pour chaque plugin, dans cet ordre. Les étapes détaillées, avec leurs options, sont dans le README de chaque plugin.

| Plugin | Bundles à déclarer dans `config/bundles.php` | Configuration (`config/packages/`) | Routes (`config/routes/`) | Migrations | Autre |
|--------|-----------------------------------------------|------------------------------------|---------------------------|------------|-------|
| agentic-feeds | `Tmonier\SyliusAgenticFeedsPlugin\TmonierSyliusAgenticFeedsPlugin` | `@TmonierSyliusAgenticFeedsPlugin/config/config.yaml` | `@TmonierSyliusAgenticFeedsPlugin/config/routes.yaml` | oui | planifier `tmonier:feeds:generate` (cron) |
| ai-catalog-copilot | `Tmonier\SyliusAiCatalogCopilotPlugin\TmonierSyliusAiCatalogCopilotPlugin` | `@TmonierSyliusAiCatalogCopilotPlugin/config/config.yaml` | `@TmonierSyliusAiCatalogCopilotPlugin/config/routes.yaml` | oui (2 tables) | installer le pont du fournisseur choisi (`symfony/ai-*-platform`) et renseigner sa clé d'API dans une variable d'environnement |
| b2b-mcp | `Tmonier\SyliusB2bMcpPlugin\TmonierSyliusB2bMcpPlugin` | `@TmonierSyliusB2bMcpPlugin/config/config.yaml` | `@TmonierSyliusB2bMcpPlugin/config/routes.yaml` | oui | `symfony/lock` (voir ci-dessous), HTTPS sur un nom d'hôte public, proxys de confiance, clé de chiffrement dédiée, `tmonier:b2b-mcp:purge` chaque jour |
| webhook-outbox | `Tmonier\SyliusWebhookOutboxPlugin\TmonierSyliusWebhookOutboxPlugin` | `@TmonierSyliusWebhookOutboxPlugin/config/config.yaml` | `@TmonierSyliusWebhookOutboxPlugin/config/routes.yaml` | oui | lancer le worker d'envoi, planifier `tmonier:webhook:purge`, définir `framework.router.default_uri` |

Total pour le pack : 4 bundles, 4 imports de configuration, 4 imports de routes, 4 jeux de migrations et 3 tâches planifiées ou processus. Trois autres bundles, ainsi que la configuration et les routes du plugin de serveur MCP, sont faits par les recettes Symfony Flex d'autres paquets (section suivante).

### Fait par les recettes d'autres paquets

Dans un projet qui utilise Symfony Flex (Sylius-Standard en utilise), `composer require` exécute les recettes des paquets dont dépendent les plugins. Elles déclarent `Symfony\AI\AiBundle\AiBundle` (`symfony/ai-bundle`), `Symfony\AI\McpBundle\McpBundle` (`symfony/mcp-bundle`) et `Sylius\McpServerPlugin\SyliusMcpServerPlugin` (`sylius/mcp-server-plugin`), et importent `@SyliusMcpServerPlugin/config/config.yaml` et `@SyliusMcpServerPlugin/config/routes.yaml`. Ne pas les répéter : importer deux fois les routes du plugin de serveur MCP fait échouer le routeur (« Do not add the mcp loader twice »). Sans recettes Flex, les ajouter à la main, comme le décrit le README du plugin MCP. L'intégration continue vérifie que les recettes ont bien été exécutées (`bin/apply-gestures --verify-flex`).

### Paquets Composer

Le limiteur de débit du plugin MCP a besoin du composant Lock de Symfony, que le plugin ne requiert pas lui-même et qu'un projet Sylius-Standard neuf ne contient pas :

```bash
composer require symfony/lock
```

### Bundles

```php
// config/bundles.php
Tmonier\SyliusAgenticFeedsPlugin\TmonierSyliusAgenticFeedsPlugin::class => ['all' => true],
Tmonier\SyliusAiCatalogCopilotPlugin\TmonierSyliusAiCatalogCopilotPlugin::class => ['all' => true],
Tmonier\SyliusB2bMcpPlugin\TmonierSyliusB2bMcpPlugin::class => ['all' => true],
Tmonier\SyliusWebhookOutboxPlugin\TmonierSyliusWebhookOutboxPlugin::class => ['all' => true],
```

### Configuration et routes

Un fichier par plugin, sous la forme ci-dessous (remplacer l'alias par celui du tableau) :

```yaml
# config/packages/tmonier_sylius_agentic_feeds.yaml
imports:
    - { resource: "@TmonierSyliusAgenticFeedsPlugin/config/config.yaml" }
```

```yaml
# config/routes/tmonier_sylius_agentic_feeds.yaml
tmonier_sylius_agentic_feeds:
    resource: "@TmonierSyliusAgenticFeedsPlugin/config/routes.yaml"
```

### Migrations

```bash
bin/console doctrine:migrations:migrate
```

## Recette Flex

Le dossier `recipe/` contient la recette Symfony Flex de ce pack, dans la structure de `symfony/recipes-contrib` : un manifeste qui déclare les bundles du tableau ci-dessus et copie les fichiers de configuration et de routes. **Elle n'est pas encore publiée.** Flex ne trouve que les recettes publiées sur `symfony/recipes-contrib` : tant qu'elle ne l'est pas, `composer require` ne déclare rien et les gestes ci-dessus restent nécessaires. Une pull request sur ce dépôt public n'est ouverte qu'avec l'accord du mainteneur.

Ce qui est testé :

- Flex applique une recette à un métapaquet : la CI sert la recette depuis un endpoint local (`bin/build-recipe-endpoint`), requiert le pack dans un projet Sylius vierge **sans appliquer les gestes**, puis rejoue les contrôles du job `install` (job `recipe`, Sylius-Standard 2.2.4 et 2.3.0).
- `bin/flex-trial` rejoue, dans un projet Symfony jetable, l'installation, le retrait (`composer remove` retire les bundles et les fichiers écrits par la recette) et le cas d'un fichier déjà présent dans l'application (il n'est pas écrasé).
- `bin/check-recipe` échoue quand la recette diffère de `etc/gestures.json` : les bundles et les fichiers viennent de la même source que les gestes.

Limite de Flex, observée à l'essai : il consigne dans `symfony.lock` les fichiers de la recette, y compris un fichier que l'application avait déjà écrit sous le même nom, et le supprime à `composer remove`. Garder un tel fichier sous un autre nom ou sous gestion de version. La recette ne lance pas les migrations et ne fait pas les étapes de la colonne « Autre ».

```bash
php tests/recipe.php               # tests de la recette et de l'endpoint
bin/check-recipe                   # recette contre etc/gestures.json
bin/flex-trial                     # rejoue l'essai (php, composer et réseau nécessaires)
bin/build-recipe-endpoint --out=DIR # le JSON que Flex lit depuis un endpoint
```

## Vérifications

`bin/check-composition` vérifie que :

- les plugins requis par `composer.json` sont exactement les quatre membres listés dans `etc/composition.json` ;
- aucun plugin exclu des packs n'apparaît dans `composer.json` ni dans un README ;
- le plugin suggéré reste dans `suggest` et hors de `require` ;
- avec `--lock`, après une résolution Composer, aucun plugin verrouillé ne requiert un plugin absent du pack (l'échec nomme la dépendance).

`bin/check-gestures` vérifie que les plugins de `etc/gestures.json` sont exactement les membres, et que chaque bundle et chaque ressource de configuration ou de routes de ce fichier figure dans les deux README, sans autre.

```bash
php tests/run.php                  # tests du vérificateur
bin/check-composition              # composition de composer.json et des README
bin/check-composition --lock=composer.lock
```

L'intégration continue résout aussi le pack dans un projet jetable (`composer update --no-install`) avec Sylius 2.2 et 2.3, et l'installe (voir ci-dessous).

## Vérifié en CI

Les gestes ci-dessus ne sont pas seulement décrits : ils sont appliqués par `bin/apply-gestures` à partir de `etc/gestures.json`, et `bin/check-gestures` échoue quand l'un des deux README décrit d'autres gestes que ce fichier.

Le job `install` crée un projet `sylius/sylius-standard` vierge (2.2.4 avec Sylius `~2.2.9`, et 2.3.0 avec Sylius `~2.3.0`), ajoute ce dépôt comme dépôt Composer de type `path` (le pack n'est pas republié pour l'essai), le requiert, applique les gestes, lance les migrations, puis `lint:container` et `doctrine:schema:validate`. Un échec nomme le plugin et l'étape : `FAIL plugin=… step=…` (`bin/step`). Un second job installe ce pack et le [pack Conformité et outillage](https://github.com/thibmonier/sylius-pack-compliance-tooling) dans le même projet et compile le conteneur (aucun conflit de service ni de route).

```bash
php tests/gestures.php             # tests des gestes, de leur contrôle contre les README et de l'enveloppe d'étape
bin/check-gestures                 # etc/gestures.json contre les deux README
bin/apply-gestures --project=DIR   # applique les gestes à un projet Symfony
```

Deux ajustements sont écrits dans le workflow plutôt que cachés. Le squelette 2.2.4 déclare `FOSRestBundle` et son fichier de configuration alors que Sylius 2.2.12 n'installe plus ce paquet : le job retire les deux quand le paquet est absent. Le schéma d'un projet Sylius-Standard vierge est lui-même désynchronisé après ses migrations (tables des bundles Mollie, de remboursement, de demande de paiement et de messenger) : `doctrine:schema:validate --skip-sync` vérifie donc le mapping, et la synchronisation n'est contrôlée que sur les tables du pack.

Ce que le job ne couvre pas : la colonne « Autre » du tableau (tâches planifiées, pont du fournisseur et clé d'API, HTTPS, worker), PHP 8.2 (Sylius-Standard exige PHP 8.3, donc le job tourne sur PHP 8.3 alors que les plugins du pack acceptent PHP 8.2) et Sylius 2.2 avant 2.2.9.

## Maintenance

Une nouvelle version majeure d'un membre demande de relire sa borne ici et de rejouer les vérifications. Un plugin qui requiert un plugin hors du pack ne peut pas y entrer.

## Licence

MIT. Voir [LICENSE](LICENSE).
