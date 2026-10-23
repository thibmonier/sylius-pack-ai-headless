# Sylius Pack: AI and headless

> Français : [README.fr.md](README.fr.md) <!-- vitrine:lang -->

[![Build](https://github.com/thibmonier/sylius-pack-ai-headless/actions/workflows/build.yaml/badge.svg)](https://github.com/thibmonier/sylius-pack-ai-headless/actions/workflows/build.yaml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

**Four Sylius 2 plugins for AI assistants and integrations, installed with one `composer require`: product feeds, an AI catalog copilot, B2B tools for the Sylius MCP server and reliable outgoing webhooks.**

This package is a Composer metapackage: it contains no code. It requires the four plugins below, with version bounds that resolve together, so that you do not choose them one by one. It is free and open source.

## What a metapackage does, and does not do

- It does: one command installs the four plugins at versions known to resolve together.
- It does not: declare the bundles, import the configuration and the routes, run the migrations or schedule the tasks. Those steps stay manual, plugin by plugin, and are listed in [What is left to do by hand](#what-is-left-to-do-by-hand). Until the Flex recipe of the pack is published on `symfony/recipes-contrib` (see [Flex recipe](#flex-recipe)), they are not done by `composer require`.
- It does not guarantee your shop: the pack is installed in a blank Sylius project by the continuous integration, with the gestures below and nothing else (see [Verified in CI](#verified-in-ci)); your own application may differ.

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
| ai-catalog-copilot | `Tmonier\SyliusAiCatalogCopilotPlugin\TmonierSyliusAiCatalogCopilotPlugin` | `@TmonierSyliusAiCatalogCopilotPlugin/config/config.yaml` | `@TmonierSyliusAiCatalogCopilotPlugin/config/routes.yaml` | yes (2 tables) | install the bridge of your provider (`symfony/ai-*-platform`) and set its API key in an environment variable |
| b2b-mcp | `Tmonier\SyliusB2bMcpPlugin\TmonierSyliusB2bMcpPlugin` | `@TmonierSyliusB2bMcpPlugin/config/config.yaml` | `@TmonierSyliusB2bMcpPlugin/config/routes.yaml` | yes | `symfony/lock` (see below), HTTPS on a public host name, trusted proxies, a dedicated encryption key, daily `tmonier:b2b-mcp:purge` |
| webhook-outbox | `Tmonier\SyliusWebhookOutboxPlugin\TmonierSyliusWebhookOutboxPlugin` | `@TmonierSyliusWebhookOutboxPlugin/config/config.yaml` | `@TmonierSyliusWebhookOutboxPlugin/config/routes.yaml` | yes | run the delivery worker, schedule `tmonier:webhook:purge`, set `framework.router.default_uri` |

Totals for the pack: 4 bundles, 4 configuration imports, 4 route imports, 4 sets of migrations and 3 scheduled tasks or processes. Three more bundles, and the configuration and routes of the MCP server plugin, are done by the Symfony Flex recipes of other packages (next section).

### Done by the recipes of other packages

In a project that uses Symfony Flex (Sylius-Standard does), `composer require` runs the recipes of the packages the plugins depend on. They register `Symfony\AI\AiBundle\AiBundle` (`symfony/ai-bundle`), `Symfony\AI\McpBundle\McpBundle` (`symfony/mcp-bundle`) and `Sylius\McpServerPlugin\SyliusMcpServerPlugin` (`sylius/mcp-server-plugin`), and import `@SyliusMcpServerPlugin/config/config.yaml` and `@SyliusMcpServerPlugin/config/routes.yaml`. Do not repeat them: importing the routes of the MCP server plugin twice makes the router fail ("Do not add the mcp loader twice"). Without Flex recipes, add them by hand, as the README of the MCP plugin describes. The continuous integration checks that the recipes ran (`bin/apply-gestures --verify-flex`).

### Composer packages

The rate limiter of the MCP plugin needs the Symfony Lock component, which the plugin does not require itself and which a new Sylius-Standard project does not contain:

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

### Migrations

```bash
bin/console doctrine:migrations:migrate
```

## Flex recipe

The `recipe/` directory holds the Symfony Flex recipe of this pack, in the layout of `symfony/recipes-contrib`: a manifest that registers the bundles of the table above and copies the configuration and routes files. **It is not published yet.** Flex only finds recipes published on `symfony/recipes-contrib`, so until then `composer require` registers nothing and the gestures above stay necessary. A pull request on that public repository is opened only with the agreement of the maintainer.

What is tested:

- Flex applies a recipe to a metapackage: the CI serves the recipe from a local endpoint (`bin/build-recipe-endpoint`), requires the pack in a blank Sylius project **without applying the gestures**, then runs the checks of the `install` job (`recipe` job, Sylius-Standard 2.2.4 and 2.3.0).
- `bin/flex-trial` replays, in a throwaway Symfony project, the installation, the removal (`composer remove` unregisters the bundles and removes the files the recipe wrote) and the case of a file already in the application (it is not overwritten).
- `bin/check-recipe` fails when the recipe differs from `etc/gestures.json`: the bundles and the files come from the same source as the gestures.

Limit of Flex, observed in the trial: it records in `symfony.lock` the files of the recipe, including one that the application had already written under the same name, and removes it on `composer remove`. Keep such a file under another name or in version control. The recipe does not run the migrations and does not do the steps of the "Other" column.

```bash
php tests/recipe.php               # tests of the recipe and of the endpoint
bin/check-recipe                   # recipe against etc/gestures.json
bin/flex-trial                     # replays the trial (php, composer and network needed)
bin/build-recipe-endpoint --out=DIR # the JSON that Flex reads from an endpoint
```

## Checks

`bin/check-composition` verifies that:

- the plugins required by `composer.json` are exactly the four members listed in `etc/composition.json`;
- no plugin excluded from the packs appears in `composer.json` or in a README;
- the suggested plugin stays in `suggest` and out of `require`;
- with `--lock`, after a Composer resolution, no locked plugin requires a plugin that is not in the pack (the failure names the dependency).

`bin/check-gestures` verifies that the plugins of `etc/gestures.json` are exactly the members, and that every bundle and every configuration or routes resource of that file is in both READMEs and that no other one is.

```bash
php tests/run.php                  # tests of the checker
bin/check-composition              # composition of composer.json and of the READMEs
bin/check-composition --lock=composer.lock
```

The continuous integration also resolves the pack in a throwaway project (`composer update --no-install`) with Sylius 2.2 and 2.3, and installs it (see below).

## Verified in CI

The gestures above are not only described: they are applied by `bin/apply-gestures` from `etc/gestures.json`, and `bin/check-gestures` fails when one of the two READMEs describes other gestures than that file.

The `install` job creates a blank `sylius/sylius-standard` project (2.2.4 with Sylius `~2.2.9`, and 2.3.0 with Sylius `~2.3.0`), adds this repository as a Composer `path` repository (the pack is not republished for the test), requires it, applies the gestures, runs the migrations, then `lint:container` and `doctrine:schema:validate`. A failure names the plugin and the step: `FAIL plugin=… step=…` (`bin/step`). A second job installs this pack and the [Compliance and tooling pack](https://github.com/thibmonier/sylius-pack-compliance-tooling) in the same project and compiles the container (no service or route conflict).

```bash
php tests/gestures.php             # tests of the gestures, of their README check and of the step wrapper
bin/check-gestures                 # etc/gestures.json against both READMEs
bin/apply-gestures --project=DIR   # applies the gestures to a Symfony project
```

Two adjustments are written in the workflow rather than hidden. The 2.2.4 skeleton declares `FOSRestBundle` and its configuration file although Sylius 2.2.12 no longer installs the package: the job removes both when the package is absent. The schema of a blank Sylius-Standard project is itself out of sync after its migrations (tables of the Mollie, refund, payment request and messenger bundles), so `doctrine:schema:validate --skip-sync` checks the mapping and the synchronisation is checked on the tables of the pack only.

What the job does not cover: the "Other" column of the table (scheduled tasks, provider bridge and API key, HTTPS, worker), PHP 8.2 (Sylius-Standard requires PHP 8.3, so the job runs on PHP 8.3 although the plugins of the pack accept PHP 8.2), and Sylius 2.2 before 2.2.9.

## Maintenance

A new major version of a member requires reviewing its bound here and running the checks again. A plugin that requires a plugin outside the pack cannot join it.

## License

MIT. See [LICENSE](LICENSE).
