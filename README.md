# Agent Skills

A collection of skills for AI coding agents. Skills are packaged instructions and scripts that extend agent capabilities.

Skills follow the [Agent Skills](https://agentskills.io/) format.

## Available Skills

### js-gnome-apps

Build native GNOME desktop applications using JavaScript (GJS) with GTK 4, Libadwaita, and the GNOME platform.

**Use when:**

- Creating, modifying, or debugging a GNOME app written in JavaScript/GJS
- Designing UI with XML or Blueprint
- GObject subclassing
- Meson build setup
- Flatpak packaging
- Tasks involving GJS bindings for GLib/GIO/GTK4/Adw libraries

### js-gnome-extensions

Build, debug, and maintain GNOME Shell extensions using GJS (GNOME JavaScript)

**Use when:**

- Creating GNOME Shell extensions
- Add UI elements like panel buttons, popup menus, quick settings toggles/sliders, or modal dialogs
- Implementing extension preferences with GTK4/Adwaita
- Debugging or testing an extension
- Porting an extension to a newer GNOME Shell version (45-49+)
- Preparing an extension for submission to [extensions.gnome.org](https://extensions.gnome.org/)
- Working with GNOME Shell internal APIs

### js-stellar-sdk

Build applications with the Stellar JS SDK.

**Use when:**

- Building applications that interact with the Stellar blockchain, including sending payments, creating accounts, issuing assets, managing trustlines, trading on the DEX, querying Horizon, interacting with Stellar RPC, streaming events, building and signing transactions, multisig, claimable balances, sponsored reserves, SEP-10 auth, federation, fee-bump transactions, and interacting with Soroban smart contracts via the JS SDK.

### js-stronghold-sdk

Integrate and build with the Stronghold Pay JS SDK and REST API for payment processing.

**Use when:**

- Working with Stronghold Pay payment integration
- Accepting ACH/bank debit payments
- Linking bank accounts
- Creating charges/tips
- Generating PayLinks
- Building checkout flows in sandbox or live environments

### rs-ratatui-crate

Build terminal user interfaces (TUIs) in Rust with Ratatui.

**Use when:**

- Setting up a new Ratatui project
- Creating or modifying terminal UI layouts
- Implementing widgets (lists, tables, charts, text, gauges, etc.)
- Handling keyboard/mouse input and events
- Structuring TUI application architecture (TEA, component-based, or monolithic patterns)
- Writing custom widgets
- Managing application state in a TUI context
- Terminal setup/teardown and panic handling
- Testing TUI rendering with TestBackend

### rs-soroban-sdk

Build smart contracts on the Stellar blockchain using the Soroban Rust SDK.

**Use when:**

- Creating Soroban smart contracts
- Implementing storage with Persistent, Temporary, or Instance storage types
- Working with auth contexts and authorization
- Handling tokens and Stellar Asset Contracts
- Writing tests with testutils
- Deploying contracts
- Working with events and logging
- Using crypto functions
- Debugging contract errors
- Security best practices and vulnerability prevention
- Avoiding common security pitfalls like missing authorization, integer overflow, or reinitialization attacks

## Installation

```bash
npx skills add https://github.com/padparadscho/skills --skill <skill-name>
```

## Usage

Skills are automatically available once installed. The agent will use them when relevant tasks are detected.

## Skill Structure

Each skill contains:

- `SKILL.md` - Instructions for the agent
- `references/` - Supporting documentation (optional)

## License

This project is licensed under the [MIT License](/LICENSE).
