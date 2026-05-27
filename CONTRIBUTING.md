# Contributing to @blull/circuitai

This repository is the **public issue tracker** for [`@blull/circuitai`](https://www.npmjs.com/package/@blull/circuitai). The source code lives in a private repository, so pull requests cannot be accepted here. Bug reports, feature requests, and design feedback through issues are very welcome.

## Filing a good issue

Please include:

1. `@blull/circuitai` version, Node version, OS.
2. A minimal reproduction. The bundled mock provider (`@blull/circuitai/testing`) keeps repros offline and deterministic:

   ```ts
   import { createMockProvider } from '@blull/circuitai/testing'
   ```
3. The full error message and, if relevant, the `RunEvent` stream from `project.stream()`.

## Security issues

For anything you suspect is a security vulnerability, please email **fabio@blull.com.br** rather than opening a public issue.

## Questions and discussion

Open an issue with the `question` label, or reach out at **fabio@blull.com.br**.
