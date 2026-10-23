# Sylius Pack : IA et headless

> English: [README.md](README.md) <!-- vitrine:lang -->

[![Build](https://github.com/thibmonier/sylius-pack-ai-headless/actions/workflows/build.yaml/badge.svg)](https://github.com/thibmonier/sylius-pack-ai-headless/actions/workflows/build.yaml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

**Quatre plugins Sylius 2 pour les assistants d'IA et les intégrations, installés avec un seul `composer require` : flux produits, copilote d'IA pour le catalogue, outils B2B pour le serveur MCP de Sylius et webhooks sortants fiables.**

Ce paquet est un métapaquet Composer : il ne contient aucun code. Il requiert les quatre plugins ci-dessous, avec des bornes de versions qui se résolvent ensemble, pour ne pas les choisir un par un. Il est gratuit et open source.

## Ce que fait un métapaquet, et ce qu'il ne fait pas

- Il fait : une commande installe les quatre plugins à des versions dont la résolution ensemble est vérifiée.
- Il ne fait pas : déclarer les bundles, importer la configuration et les routes, lancer les migrations ni planifier les tâches. Ces étapes restent manuelles, plugin par plugin, et sont listées dans [Ce qui reste à faire à la main](#ce-qui-reste-à-faire-à-la-main). Aucune recette Symfony Flex n'existe encore pour ce pack.
- Il ne garantit pas une boutique qui démarre : la composition est vérifiée par la résolution de Composer (voir [Vérifications](#vérifications)), pas par une installation complète.

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
| ai-catalog-copilot | `Symfony\AI\AiBundle\AiBundle` et `Tmonier\SyliusAiCatalogCopilotPlugin\TmonierSyliusAiCatalogCopilotPlugin` | `@TmonierSyliusAiCatalogCopilotPlugin/config/config.yaml` | `@TmonierSyliusAiCatalogCopilotPlugin/config/routes.yaml` | oui (2 tables) | installer le pont du fournisseur choisi (`symfony/ai-*-platform`) et renseigner sa clé d'API dans une variable d'environnement |
| b2b-mcp | `Symfony\AI\McpBundle\McpBundle`, `Sylius\McpServerPlugin\SyliusMcpServerPlugin` et `Tmonier\SyliusB2bMcpPlugin\TmonierSyliusB2bMcpPlugin` | `@SyliusMcpServerPlugin/config/config.yaml` et `@TmonierSyliusB2bMcpPlugin/config/config.yaml` | `@SyliusMcpServerPlugin/config/routes.yaml` et `@TmonierSyliusB2bMcpPlugin/config/routes.yaml` | oui | HTTPS sur un nom d'hôte public, proxys de confiance, clé de chiffrement dédiée, `tmonier:b2b-mcp:purge` chaque jour |
| webhook-outbox | `Tmonier\SyliusWebhookOutboxPlugin\TmonierSyliusWebhookOutboxPlugin` | `@TmonierSyliusWebhookOutboxPlugin/config/config.yaml` | `@TmonierSyliusWebhookOutboxPlugin/config/routes.yaml` | oui | lancer le worker d'envoi, planifier `tmonier:webhook:purge`, définir `framework.router.default_uri` |

Total pour le pack : 7 bundles (certains peuvent déjà être déclarés dans votre boutique), 5 imports de configuration, 5 imports de routes, 4 jeux de migrations et 3 tâches planifiées ou processus.

### Bundles

```php
// config/bundles.php
Symfony\AI\AiBundle\AiBundle::class => ['all' => true],
Symfony\AI\McpBundle\McpBundle::class => ['all' => true],
Sylius\McpServerPlugin\SyliusMcpServerPlugin::class => ['all' => true],
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

Le plugin MCP importe la configuration et les routes de `SyliusMcpServerPlugin` et de son propre plugin : voir son README.

### Migrations

```bash
bin/console doctrine:migrations:migrate
```

## Vérifications

`bin/check-composition` vérifie que :

- les plugins requis par `composer.json` sont exactement les quatre membres listés dans `etc/composition.json` ;
- aucun plugin exclu des packs n'apparaît dans `composer.json` ni dans un README ;
- le plugin suggéré reste dans `suggest` et hors de `require` ;
- avec `--lock`, après une résolution Composer, aucun plugin verrouillé ne requiert un plugin absent du pack (l'échec nomme la dépendance).

```bash
php tests/run.php                  # tests du vérificateur
bin/check-composition              # composition de composer.json et des README
bin/check-composition --lock=composer.lock
```

L'intégration continue résout aussi le pack dans un projet jetable (`composer update --no-install`) avec Sylius 2.2 et 2.3.

## Maintenance

Une nouvelle version majeure d'un membre demande de relire sa borne ici et de rejouer les vérifications. Un plugin qui requiert un plugin hors du pack ne peut pas y entrer.

## Licence

MIT. Voir [LICENSE](LICENSE).
