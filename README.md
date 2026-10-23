# Sylius Pack: AI and headless

> Français : [README.fr.md](README.fr.md) <!-- vitrine:lang -->

[![Build](https://github.com/thibmonier/sylius-pack-ai-headless/actions/workflows/build.yaml/badge.svg)](https://github.com/thibmonier/sylius-pack-ai-headless/actions/workflows/build.yaml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

**Four Sylius 2 plugins for AI assistants and integrations, installed with one `composer require`: product feeds, an AI catalog copilot, B2B tools for the Sylius MCP server and reliable outgoing webhooks.**

This package is a Composer metapackage: it contains no code. It requires the four plugins below, with version bounds that resolve together, so that you do not choose them one by one. It is free and open source.

## What a metapackage does, and does not do

- It does: one command installs the four plugins at versions known to resolve together.
- It does not: declare the bundles, import the configuration and the routes, run the migrations or schedule the tasks. Those steps stay manual, plugin by plugin, and are listed in [What is left to do by hand](#what-is-left-to-do-by-hand). No Symfony Flex recipe exists yet for this pack.
- It does not guarantee a running shop: the composition is checked by Composer resolution (see [Checks](#checks)), not by a full installation.

## Requirements

- PHP 8.2 or later
- Sylius 2.2.9 or later (2.3 supported)
- `symfony/ai-bundle` and `symfony/ai-platform` `^0.14` (required by the AI catalog copilot): the pack cannot be installed next to a package that needs another minor version of Symfony AI.

## Installation

```bash
composer require tmonier/sylius-pack-ai-headless
```

Then follow [What is left to do by hand](#what-is-left-to-do-by-hand).

## Plugins in the pack

| Plugin | Bound | Role |
|--------|-------|------|
| [sylius-agentic-feeds-plugin](https://github.com/thibmonier/sylius-agentic-feeds-plugin) | `^1.0` | Product feeds (Google Merchant Center, OpenAI, Stripe) and Product JSON-LD |
| [sylius-ai-catalog-copilot-plugin](https://github.com/thibmonier/sylius-ai-catalog-copilot-plugin) | `^1.0` | Reviewable product description, meta and translation suggestions through Symfony AI |
| [sylius-b2b-mcp-plugin](https://github.com/thibmonier/sylius-b2b-mcp-plugin) | `^1.2` | B2B tools for the official Sylius MCP server (OAuth 2.1 with PKCE) |
| [sylius-webhook-outbox-plugin](https://github.com/thibmonier/sylius-webhook-outbox-plugin) | `^1.0` | Outgoing webhooks with a transactional outbox, signatures, retries and replay |

Suggested, not installed (`suggest`):

| Plugin | Why |
|--------|-----|
| [sylius-audit-log-plugin](https://github.com/thibmonier/sylius-audit-log-plugin) | The copilot records the product change made when a suggestion is accepted, and the MCP tool calls are mirrored into the journal. It is part of the [Compliance and tooling pack](https://github.com/thibmonier/sylius-pack-compliance-tooling). |

## What is left to do by hand

For each plugin, in this order. The detailed steps, with their options, are in the README of each plugin.

| Plugin | Bundles to register in `config/bundles.php` | Configuration (`config/packages/`) | Routes (`config/routes/`) | Migrations | Other |
|--------|----------------------------------------------|------------------------------------|---------------------------|------------|-------|
| agentic-feeds | `Tmonier\SyliusAgenticFeedsPlugin\TmonierSyliusAgenticFeedsPlugin` | `@TmonierSyliusAgenticFeedsPlugin/config/config.yaml` | `@TmonierSyliusAgenticFeedsPlugin/config/routes.yaml` | yes | schedule `tmonier:feeds:generate` (cron) |
| ai-catalog-copilot | `Symfony\AI\AiBundle\AiBundle` and `Tmonier\SyliusAiCatalogCopilotPlugin\TmonierSyliusAiCatalogCopilotPlugin` | `@TmonierSyliusAiCatalogCopilotPlugin/config/config.yaml` | `@TmonierSyliusAiCatalogCopilotPlugin/config/routes.yaml` | yes (2 tables) | install the bridge of your provider (`symfony/ai-*-platform`) and set its API key in an environment variable |
| b2b-mcp | `Symfony\AI\McpBundle\McpBundle`, `Sylius\McpServerPlugin\SyliusMcpServerPlugin` and `Tmonier\SyliusB2bMcpPlugin\TmonierSyliusB2bMcpPlugin` | `@SyliusMcpServerPlugin/config/config.yaml` and `@TmonierSyliusB2bMcpPlugin/config/config.yaml` | `@SyliusMcpServerPlugin/config/routes.yaml` and `@TmonierSyliusB2bMcpPlugin/config/routes.yaml` | yes | HTTPS on a public host name, trusted proxies, a dedicated encryption key, daily `tmonier:b2b-mcp:purge` |
| webhook-outbox | `Tmonier\SyliusWebhookOutboxPlugin\TmonierSyliusWebhookOutboxPlugin` | `@TmonierSyliusWebhookOutboxPlugin/config/config.yaml` | `@TmonierSyliusWebhookOutboxPlugin/config/routes.yaml` | yes | run the delivery worker, schedule `tmonier:webhook:purge`, set `framework.router.default_uri` |

Totals for the pack: 7 bundles (some may already be registered in your shop), 5 configuration imports, 5 route imports, 4 sets of migrations and 3 scheduled tasks or processes.

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

### Configuration and routes

One file per plugin, in the form below (replace the alias with the one of the table):

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

The MCP plugin imports the configuration and the routes of both `SyliusMcpServerPlugin` and its own plugin: see its README.

### Migrations

```bash
bin/console doctrine:migrations:migrate
```

## Checks

`bin/check-composition` verifies that:

- the plugins required by `composer.json` are exactly the four members listed in `etc/composition.json`;
- no plugin excluded from the packs appears in `composer.json` or in a README;
- the suggested plugin stays in `suggest` and out of `require`;
- with `--lock`, after a Composer resolution, no locked plugin requires a plugin that is not in the pack (the failure names the dependency).

```bash
php tests/run.php                  # tests of the checker
bin/check-composition              # composition of composer.json and of the READMEs
bin/check-composition --lock=composer.lock
```

The continuous integration also resolves the pack in a throwaway project (`composer update --no-install`) with Sylius 2.2 and 2.3.

## Maintenance

A new major version of a member requires reviewing its bound here and running the checks again. A plugin that requires a plugin outside the pack cannot join it.

## License

MIT. See [LICENSE](LICENSE).
