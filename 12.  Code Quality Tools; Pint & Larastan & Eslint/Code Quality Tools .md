# Code Quality Tools: Pint & Larastan & Eslint

- [Concept Intoduction](#concept-intoduction)
- [Differences](#differences)
- [Laravel Pint](#laravel-pint)
  - [Overview](#overview)
  - [Key Commands](#key-commands)
  - [Configuration (pint.json)](#configuration-pintjson)
  - [Presets](#presets)
  - [Rules](#rules)
  - [Notes](#notes)
- [Larastan](#larastan)
  - [Overview](#overview-1)
  - [Key Commands](#key-commands-1)
  - [Configuration (phpstan.neon)](#configuration-phpstanneon)
  - [Strictness Levels](#strictness-levels)
  - [Features](#features)
  - [Notes](#notes-1)
- [ESLint with Vue Plugin](#eslint-with-vue-plugin-quick-notes)
  - [Overview](#overview-2)
  - [Installation](#installation-1)
  - [Key Commands](#key-commands-2)
  - [Configuration (.eslintrc.json)](#configuration-eslintrcjson)
  - [Rules](#rules-1)
  - [Notes](#notes-2)


## Concept Intoduction

Code quality tools help maintain clean, reliable, and consistent codebases. In Laravel, **Pint** and **Larastan** are two essential tools that serve distinct purposes:

- **Pint**: Focuses on *code style* (formatting, aesthetics).
- **Larastan**: Focuses on *code correctness* (finding bugs, type errors).  
  Together, they ensure your code is both visually consistent and functionally sound.

### ==> *<u>Use Pint to make code pretty, Larastan to make it reliable.</u>*

## Differences

| Aspect       | Laravel Pint                                                             | Larastan                                                                                  |
| ------------ | ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------- |
| **Purpose**  | Fixes code style (e.g., indentation, braces) for consistency.            | Detects logical errors (e.g., type mismatches, undefined properties) via static analysis. |
| **Function** | Reformats code to match style rules.                                     | Analyzes code without running it; doesn’t modify files.                                   |
| **Example**  | Fixes `function index(){return 1;}` to `function index() { return 1; }`. | Flags `$user->email_address` if `email_address` doesn’t exist.                            |
| **Config**   | `pint.json` for style presets/rules.                                     | `phpstan.neon` for analysis paths/levels.                                                 |
| **CI Use**   | Enforces style standards (e.g., `--test`).                               | Catches bugs early (e.g., failing builds on errors).                                      |

**Why both?** Use Pint for readable, standardized code and Larastan to catch bugs before they hit production. They complement each other for a robust codebase.

## Laravel Pint

### Overview

- **PHP code style fixer** built on PHP CS Fixer.
- Ensures consistent, opinionated code style.
- **Included by default** in new Laravel projects.

### Key Commands

- Fix code style:
  
  ```bash
  ./vendor/bin/pint
  ```

- Fix specific files/directories:
  
  ```bash
  ./vendor/bin/pint app/Models
  ```

- `-v`: Show detailed changes.

- `--test`: Check style errors without fixing (non-zero exit code if errors).

- `--diff=[branch]`: Fix files changed vs. a branch (e.g., `main`).

- `--dirty`: Fix uncommitted changes.

- `--repair`: Fix errors but exit with non-zero code if fixes made.

### Configuration (pint.json)

- Optional; create `pint.json` in project root:
  
  ```json
  {
      "preset": "laravel",
      "rules": {
          "simplified_null_return": true
      }
  }
  ```

- Use custom config:
  
  ```bash
  ./vendor/bin/pint --config path/to/pint.json
  ```

### Presets

- Predefined style rule sets.

- **Default**: `laravel` (Laravel’s style).

- Others: `psr12`, `per`, `symfony`, `empty`.

- Set via command:
  
  ```bash
  ./vendor/bin/pint --preset psr12
  ```

- Or in `pint.json`:
  
  ```json
  {
      "preset": "psr12"
  }
  ```

### Rules

- Customize style guidelines in `pint.json`:
  
  ```json
  {
      "preset": "laravel",
      "rules": {
          "array_indentation": false
      }
  }
  ```

- Uses PHP CS Fixer rules.

### Notes

- No config needed for default style.
- Use `--test` in CI for style checks.
- Excludes `vendor` directory by default.

## Larastan

### Overview

- **Static analysis tool** for Laravel, built on PHPStan.
- Finds bugs/type errors without running code.
- Understands Laravel’s magic (e.g., Eloquent, facades).
- **Not included** in Laravel; requires installation.

### Installation

- Install via Composer:
  
  ```bash
  composer require --dev "larastan/larastan"
  ```

### Key Commands

- Run analysis:
  
  ```bash
  ./vendor/bin/phpstan analyse
  ```

- Specify directories:
  
  ```bash
  ./vendor/bin/phpstan analyse app
  ```

- Set strictness (0–9, higher = stricter):
  
  ```bash
  ./vendor/bin/phpstan analyse -l 6
  ```

- Generate baseline to ignore existing errors:
  
  ```bash
  ./vendor/bin/phpstan analyse --generate-baseline
  ```

- Increase memory:
  
  ```bash
  ./vendor/bin/phpstan analyse --memory-limit=2G
  ```

### Configuration (phpstan.neon)

- Create `phpstan.neon` in project root:
  
  ```neon
  includes:
      - ./vendor/nunomaduro/larastan/extension.neon
  parameters:
      paths:
          - app
      level: 6
      ignoreErrors:
          - '#Unknown property App\\Models\\User::email_address#'
  ```

- Customize paths, strictness, or suppress errors.

### Strictness Levels

- **0**: Basic checks (e.g., unknown classes).
- **9**: Strict typing, nullable checks.
- Recommended: Start at level 5–6.

### Features

- Detects type mismatches, undefined properties, invalid method calls.
- Supports Laravel-specific features (Eloquent, facades).
- Checks Laravel Octane compatibility.

### Notes

- Requires **PHP 8.0+**, **Laravel 9+**.
- Use in CI to catch bugs.
- May flag Laravel’s dynamic features; use `ignoreErrors` to suppress.
- Focuses on correctness, not formatting.

## ESLint with Vue Plugin

### Overview

- **Code style linter and fixer** for JavaScript and Vue.js, built on ESLint with `eslint-plugin-vue`.
- Ensures consistent code style (e.g., indentation, quotes) in `.js` and `.vue` files.
- Supports Vue Single File Components (SFCs) in Laravel + Inertia projects.
- Uses **Airbnb** style rules for strict, industry-standard formatting.
- **Not included** in Laravel; requires installation.

### Installation

- Install ESLint, Vue plugin, and Airbnb rules via npm (run in project root):
  
  ```bash
  npm install --save-dev eslint eslint-plugin-vue eslint-config-airbnb-base eslint-plugin-import
  ```

- Initialize ESLint with Vue and Airbnb:
  
  ```bash
  npx eslint --init
  ```
  
  - Choose: “To check syntax and find problems,” JavaScript modules, Vue.js, no TypeScript, Node/Browser, Airbnb style, JSON config.

### Key Commands

- Fix code style in `resources/js` (for `.js` and `.vue` files):
  
  ```bash
  npx eslint --fix resources/js
  ```

- Check for style errors without fixing (non-zero exit code if errors):
  
  ```bash
  npx eslint resources/js
  ```

- Lint specific file or directory:
  
  ```bash
  npx eslint resources/js/Pages/Home.vue
  ```

### Configuration (.eslintrc.json)

- Create `.eslintrc.json` in project root:
  
  ```json
  {
      "env": {
          "browser": true,
          "es2021": true,
          "node": true
      },
      "extends": [
          "airbnb-base",
          "plugin:vue/vue3-essential"
      ],
      "parserOptions": {
          "ecmaVersion": 12,
          "sourceType": "module"
      },
      "plugins": [
          "vue"
      ],
      "rules": {
          "vue/multi-word-component-names": "off"
      }
  }
  ```

- **Path**: ESLint targets `resources/js` for Vue files in Laravel + Inertia.

- **Custom rules**: Disable `vue/multi-word-component-names` for Inertia’s single-word page names (e.g., `Home.vue`).

- **Airbnb**: Enforces strict style (e.g., single quotes, no trailing commas).

### Rules

- **Airbnb base**: Industry-standard rules for JavaScript (e.g., `import/no-unresolved`, `no-unused-vars`).

- **Vue plugin**: Adds Vue-specific rules (e.g., `vue/no-v-html`, `vue/valid-v-bind`).

- Customize rules in `.eslintrc.json`:
  
  ```json
  {
      "rules": {
          "no-console": "warn",
          "vue/no-unused-components": "error"
      }
  }
  ```

### Notes

- **Path**: Configured for `resources/js` (Laravel + Inertia’s Vue folder).
- **JavaScript only**: No TypeScript, as per your preference.
- **CI use**: Run `npx eslint resources/js` to catch style errors.
- **Complements Pint**: ESLint formats Vue/JS, while Pint handles PHP.
- Works with Laravel Mix or Vite (Inertia’s build tools).
- Use with Prettier for formatting if desired (separate setup).
