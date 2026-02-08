# Agent Skills

A collection of skills for AI coding agents. Skills are packaged instructions and scripts that extend agent capabilities.

Skills follow the [Agent Skills](https://agentskills.io/) format.

## Available Skills

- [js-gnome-apps](/skills/js-gnome-apps/SKILL.md)

Develop native GNOME desktop applications using JavaScript (GJS) with GTK 4, Libadwaita, and the GNOME platform libraries. Covers UI design with XML and Blueprint, GObject subclassing, Meson build configuration, GSettings integration, Flatpak packaging, and GJS bindings for GLib, GIO, GTK4, and Adwaita.

- [js-gnome-extensions](/skills/js-gnome-extensions/SKILL.md)

Develop, debug, and maintain GNOME Shell extensions using GJS with ESModules. Covers extension architecture (metadata.json, extension.js, prefs.js), UI components (panel buttons, popup menus, quick settings, dialogs), GTK4/Adwaita preferences, notifications, search providers, and integration with GNOME Shell internal APIs (Clutter, St, Meta, Shell).

- [js-stellar-sdk](/skills/js-stellar-sdk/SKILL.md)

Develop blockchain applications using the Stellar JavaScript SDK (@stellar/stellar-sdk). Covers account and transaction management, payment operations, asset issuance, trustlines, DEX trading, Horizon API queries, Stellar RPC integration, event streaming, multisig workflows, claimable balances, sponsored reserves, SEP-10 authentication, and Soroban smart contract interactions.

- [js-stronghold-sdk](/skills/js-stronghold-sdk/SKILL.md)

Integrate payment processing using the Stronghold Pay JavaScript SDK and REST API. Covers ACH and bank debit payment acceptance, bank account linking, customer and payment source management, charge and tip creation, PayLink hosted payment pages, and drop-in UI components for both sandbox and live environments.

- [rs-ratatui-crate](/skills/rs-ratatui-crate/SKILL.md)

Create interactive terminal user interfaces (TUIs) in Rust using the Ratatui crate. Covers immediate-mode rendering, layout systems, built-in widgets (lists, tables, charts, gauges, text), custom widget development, keyboard and mouse input handling, application architecture patterns (TEA, component-based, monolithic), state management, and terminal lifecycle management.

- [rs-soroban-sdk](/skills/rs-soroban-sdk/SKILL.md)

Develop smart contracts for the Stellar blockchain using the Soroban Rust SDK. Covers contract structure and deployment, storage types (Persistent, Temporary, Instance), authorization and auth contexts, token handling with Stellar Asset Contracts, testing with testutils, event logging, cryptographic functions, error handling, and security best practices to prevent common vulnerabilities.

- [rs-yew-crate](/skills/rs-yew-crate/SKILL.md)

Develop frontend web applications using Rust and WebAssembly with the Yew framework. Covers function components, hooks, props, the html! macro, routing with yew-router, state management, context API, event handling, data fetching, server-side rendering, Suspense, and integration with the WASM ecosystem (gloo, wasm-bindgen, web-sys).

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
