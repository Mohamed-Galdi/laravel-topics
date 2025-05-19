# Code Quality Tools: Pint & Larastan
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
- [Final Notes](#final-notes)

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
  composer require nunomaduro/larastan --dev
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

## Final Notes

- **Run both** in CI: Pint for style, Larastan for correctness.
- Pint is pre-installed; Larastan needs setup.
- Use Pint to make code pretty, Larastan to make it reliable.
