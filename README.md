# Atmosphere Copilot Plugins

Copilot CLI plugins for the [Atmosphere](https://github.com/vexxhost/atmosphere) private cloud platform — ships skills covering development workflow, contribution guidelines, and Helm chart management.

## Prerequisites

- [GitHub Copilot CLI](https://docs.github.com/en/copilot/how-tos/set-up/install-copilot-cli) installed
- An active [Copilot subscription](https://github.com/features/copilot/plans)

## Installation

### GitHub Copilot CLI

```
/plugin marketplace add ricolin/copilot-plugins
/plugin install atmosphere@ricolin-copilot-plugins
```

Verify the plugin is loaded:

```
/skills list
```

### Gemini CLI

You can install the atmosphere skill directly using the Gemini CLI:

```bash
gemini skills install https://github.com/ricolin/copilot-plugins.git --path plugins/atmosphere/skills/atmosphere
```

Or install the repository as an extension (which bundles the skill):

```bash
gemini extensions install https://github.com/ricolin/copilot-plugins.git
```

### Claude Code

```
claude plugin add ricolin/copilot-plugins
claude plugin install atmosphere
```

You should see the `atmosphere` skill listed after installation.

## Usage

### `atmosphere` plugin

The atmosphere plugin ships development workflow skills for contributing to the Atmosphere project:

- **atmosphere** — Development guidelines covering conventional commits, DCO sign-off, Helm chart vendor + patch workflow, release notes with reno, Ansible role defaults formatting, and PR conventions.
