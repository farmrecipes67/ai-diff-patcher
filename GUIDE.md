# ai-diff-patcher

[![npm](https://img.shields.io/badge/npm-vX.X.X-blue)](https://www.npmjs.com/package/ai-diff-patcher)  
[![build](https://img.shields.io/badge/build-passing-brightgreen)](https://your-ci.example.com)  
[![coverage](https://img.shields.io/badge/coverage-XX%25-yellow)](https://your-coverage.example.com)  
[![license](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)

AI-powered structured code diffs with safe patching and rollback support. Use modern LLMs to generate semantically-aware diffs, preview proposed changes, and apply them safely with automatic backup and rollback functionality.

- Generate structured diffs (not just textual patches)
- Preview and validate patches before applying
- Safe apply with backups and optional automatic commits
- Full rollback of applied changes
- Provider-agnostic: OpenAI, Anthropic/Claude, Google Gemini (and others)

---

## Table of contents

- [Install](#install)
- [Quick start](#quick-start)
  - [OpenAI example](#openai-example)
  - [Anthropic / Claude example](#anthropic--claude-example)
  - [Google Gemini example](#google-gemini-example)
- [Advanced usage](#advanced-usage)
- [API reference](#api-reference)
- [Events](#events)
- [Best practices & security](#best-practices--security)
- [Contributing](#contributing)
- [License](#license)

---

## Install

Install from npm:

```bash
npm install ai-diff-patcher
```

Node.js 16+ recommended.

---

## Quick start

Basic ideas:
- Provide the files you want to compare or modify as structured objects.
- Ask the library to generate a structured diff using your chosen LLM provider.
- Preview the patch, then apply it safely (backups created).
- Roll back if needed.

Important: API keys are provided by you (e.g., via environment variables). This package never hardcodes provider credentials.

### OpenAI example

```js
import { DiffPatcher } from 'ai-diff-patcher';

// Provide your OpenAI API key via environment variable (or other secret manager).
// NEVER hardcode credentials in source code.
const patcher = new DiffPatcher({
  provider: 'openai',
  apiKey: process.env.OPENAI_API_KEY,
  model: 'gpt-4o-mini',       // optional; provider/model specific
  temperature: 0.2,           // optional LLM tuning
});

const originalFiles = [
  { path: 'src/index.js', content: "export default function sum(a, b) {return a + b}\n" },
];

const modifiedFiles = [
  { path: 'src/index.js', content: "export default function sum(a, b) { /* validate */ if (!Number.isFinite(a) || !Number.isFinite(b)) throw new TypeError('args must be numbers'); return a + b; }\n" },
];

async function run() {
  // Generate a structured diff
  const diff = await patcher.generateDiff({ originalFiles, modifiedFiles, purpose: 'Add input validation to sum' });

  // Preview the patch in human-readable form
  const preview = await patcher.previewPatch(diff);
  console.log('Preview:', preview.humanReadable);

  // Apply the patch to the filesystem (creates backups)
  const result = await patcher.applyPatch({ diff, targetDir: process.cwd(), createBackup: true, commit: true });
  console.log('Applied:', result.changeId);

  // If needed, rollback
  // await patcher.rollback({ changeId: result.changeId });
}

run().catch(console.error);
```

### Anthropic / Claude example

```js
import { DiffPatcher } from 'ai-diff-patcher';

const patcher = new DiffPatcher({
  provider: 'claude',               // anthopic/Claude provider
  apiKey: process.env.CLAUDE_API_KEY,
  model: 'claude-2.1-100k',
  temperature: 0.0,
});

const diffs = await patcher.generateDiff({
  originalFiles: [{ path: 'app.js', content: '...' }],
  modifiedFiles: [{ path: 'app.js', content: '... changed ...' }],
  purpose: 'Refactor error handling',
});

console.log(await patcher.previewPatch(diffs));
```

### Google Gemini example

```js
import { DiffPatcher } from 'ai-diff-patcher';

const patcher = new DiffPatcher({
  provider: 'gemini',
  apiKey: process.env.GEMINI_API_KEY,
  model: 'gemini-pro',
});

const diff = await patcher.generateDiff({
  originalFiles: [{ path: 'lib/util.ts', content: '...' }],
  modifiedFiles: [{ path: 'lib/util.ts', content: '...' }],
  purpose: 'Simplify utility functions',
});

console.log(diff.summary);
await patcher.applyPatch({ diff, targetDir: './', createBackup: true });
```

---

## Advanced usage

- Dry-run only: do not create backups or write to disk; useful for CI checks.
- Auto-commit: optionally create a git commit after applying changes.
- Custom validation hooks: register validators to check semantic or linting constraints before apply.
- Partial apply: choose which files/hunks from a structured diff to apply.

Example with dry-run and validators:

```js
const patcher = new DiffPatcher({ provider: 'openai', apiKey: process.env.OPENAI_API_KEY });

const diff = await patcher.generateDiff({ originalFiles, modifiedFiles });
const preview = await patcher.previewPatch(diff);

// Validate (custom)
const validation = await patcher.runValidators(diff, [
  async (diff) => { /* run linter or tests */ return { ok: true }; }
]);

if (validation.every(v => v.ok)) {
  const result = await patcher.applyPatch({ diff, targetDir: './', dryRun: true });
  console.log('Dry-run passed, no files written');
} else {
  console.error('Validation failed:', validation);
}
```

---

## API reference

Note: The following is a concise reference for the public API. Types shown are illustrative; check TypeScript definitions in the package for exact shapes.

Top-level exports:
- DiffPatcher (class)

DiffPatcher constructor:
- new DiffPatcher(options)
  - options.provider: string — 'openai' | 'claude' | 'gemini' | custom
  - options.apiKey: string — API key for the chosen provider (must be supplied by caller; never stored in repo)
  - options.model?: string — model id to use
  - options.temperature?: number — LLM temperature
  - options.timeoutMs?: number — request timeout
  - options.baseUrl?: string — optional override for provider endpoint (for proxies/self-hosting)

Core methods:
- generateDiff({ originalFiles, modifiedFiles, purpose?, metadata? }) => Promise<DiffResult>
  - originalFiles / modifiedFiles: Array<FileObject>
    - FileObject: { path: string, content: string }
  - purpose?: string — high-level instruction to the LLM
  - metadata?: object — opaque metadata

- previewPatch(diff, options?) => Promise<PatchPreview>
  - PatchPreview: { humanReadable: string, files: Array<{ path, diffHunks }>, summary: string }

- applyPatch({ diff, targetDir, createBackup = true, backupDir?, dryRun = false, commit = false, commitMessage? }) => Promise<ApplyResult>
  - Writes the selected changes to disk (unless dryRun).
  - createBackup true: before writing, a timestamped backup is made.
  - commit true: optionally creates a git commit (if repo present and user config permits).
  - ApplyResult: { changeId: string, appliedFiles: string[], backupPath?: string }

- rollback({ changeId, restoreTo? }) => Promise<RollbackResult>
  - Restores files from the recorded backup for the given changeId.
  - RollbackResult: { changeId: string, restoredFiles: string[] }

- listChanges() => Promise<Array<{ changeId, timestamp, summary }>>
  - List of applied changes and their metadata.

- runValidators(diff, validators: Array<Function>) => Promise<Array<ValidationResult>>
  - Validators can be async, return { ok: boolean, reason?: string }

Events (see below):
- on(eventName, handler)
- off(eventName, handler)

Errors:
- Throws exceptions derived from Error with codes:
  - PatcherError.BAD_INPUT
  - PatcherError.LLM_ERROR
  - PatcherError.IO_ERROR
  - PatcherError.VALIDATION_FAILED
  - PatcherError.TIMEOUT

Types (high level):
- DiffResult: { id, summary, files: Array<{ path, hunks: Array<Hunk> }>, metadata }
- Hunk: { type: 'add'|'modify'|'delete', patchText: string, structuredChange: object }
- PatchPreview: { humanReadable, files, summary }

Provider specifics:
- The apiKey passed in constructor is forwarded to provider SDKs or HTTP endpoints. You may also supply provider-specific fields (e.g., model, organization).
- For sensitive environments, prefer a secret manager or environment variables.

---

## Events

DiffPatcher implements a simple event emitter API:

- 'progress' => handler({ step: string, percent?: number, message?: string })
- 'info' => handler({ message: string })
- 'warning' => handler({ message: string, code?: string })
- 'error' => handler(error)

Example:

```js
patcher.on('progress', p => console.log('Progress:', p));
patcher.on('warning', w => console.warn('Warning:', w));
```

---

## Best practices & security

- Pass API keys and secrets via environment variables, secret managers, or runtime configuration; never commit them to source control.
- This library never hardcodes provider credentials. You are responsible for supplying them when creating the DiffPatcher instance.
- Use dry runs and validators in CI to reduce the risk of undesired changes.
- Backups are created by default when applying patches; keep backup retention and storage policy aligned with your project's security posture.
- Review human-readable previews and automated tests before automatically committing changes.

---

## Contributing

Contributions, bug reports, and pull requests are welcome.

- Fork the repo
- Create a branch for your feature/fix
- Write tests and update documentation
- Open a pull request describing your change

Check the repository's issue tracker for guidance on feature requests and roadmap.

---

## License

MIT License — see LICENSE file for details.

Copyright (c) YYYY Your Name or Organization

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files...
(Full MIT text included in the LICENSE file.)

---

If you have questions or need examples for your specific provider integration, tell me which provider and use case (CI, pre-commit hook, or interactive CLI) and I can provide tailor-made examples.