# Security Guidelines & Operating Protocol for AI Coding Agents

> **Objective:** This document defines mandatory security constraints, package management policies, and operational boundaries for AI coding agents (Claude, Cursor, Copilot, Docker Sandboxes, etc.). All automated agents working in this repository must strictly adhere to these rules before executing commands, installing dependencies, or modifying configuration files.

---

## 1. Zero-Trust Dependency Management

Agents must treat all third-party dependencies as untrusted code. Automated package updates and loose semver ranges are the primary vectors for supply-chain compromise.

### 1.1 Strict Version Pinning (No Carets or Tildes)
* **RULE:** Agents **MUST NOT** use `^` (caret) or `~` (tilde) when adding or updating dependencies in `package.json`.
* **RULE:** Every dependency version **MUST** be pinned to an exact string (e.g., `"axios": "1.13.5"`, never `"^1.13.5"`).
* **RULE:** Agents **MUST NOT** run blanket update commands (`npm update`, `pnpm update`, `npm-check-updates`) across all packages. Dependencies must be updated deliberately one at a time.

```json
/* INCORRECT (Vulnerable to zero-day resolution) */
"dependencies": {
  "axios": "^1.13.2",
  "react": "^19.2.0"
}

/* CORRECT (Strictly pinned) */
"dependencies": {
  "axios": "1.13.5",
  "react": "19.2.1"
}
```

### 1.2 Mandatory Script Execution Prevention
* **RULE:** Lifecycle install scripts (`preinstall`, `install`, `postinstall`, `prepare`) **MUST NEVER** execute automatically during installation.
* Every project root **MUST** contain a `.npmrc` enforcing script and Git-dependency restrictions:

```ini
# .npmrc
ignore-scripts=true
allow-git=none
```

* If a package requires native binaries or compilation (e.g., `esbuild`, `sharp`):
  1. The agent must verify the package is authentic.
  2. The agent must document the requirement.
  3. The build step must be run explicitly for only that package (`npm rebuild <pkg>`) rather than enabling scripts globally.

### 1.3 Block Git-Based Dependencies
* **RULE:** Dependencies declared as arbitrary Git URLs (e.g., `git+https://...`, `github:org/repo`) are strictly prohibited unless explicitly whitelisted by the user.
* Enforce `allow-git=none` (or `allow-git=root`) in `.npmrc`.

### 1.4 Package Cooldown & Release Age
* **RULE:** Agents **MUST NOT** install any package version released within the last **72 hours** (minimum 24 hours).
* If using `pnpm`, enforce:
  ```ini
  minimum-release-age=1440
  ```
* Most malicious packages or compromised releases are identified, reported, and yanked from registries within 24 to 72 hours. Never adopt day-zero package releases.

---

## 2. Pre-Install Vetting & Gatekeeping

Before installing any new package, the agent must perform the following vetting checklist:

1. **Evaluate Native Alternatives (Micro-package Ban):**
   * Do not install third-party packages for trivial logic (e.g., `is-number`, `left-pad`, simple array flattening, basic string manipulation).
   * Write standard TypeScript/JavaScript natively instead of expanding the attack surface.
2. **Check Package Metrics:**
   * Package must have an established history (> 1 year active).
   * Verifiable GitHub repository matching the npm package.
   * High download volume and multiple active maintainers.
3. **Use Behavioral Scanners:**
   * When installing in the sandbox, use security scanning tools like **Socket** or **npq**:
     ```bash
     npx @socketsecurity/cli npm install <package>@<pinned-version> --ignore-scripts
     ```
   * Flag and abort if the scanner detects:
     * Network calls during install
     * File system access outside the project tree
     * Shell execution or obfuscated base64 strings
     * Typosquatting / similarity to other popular packages

---

## 3. Lockfile Integrity & Git Hygiene

Lockfiles are critical security boundaries. Tampering with lockfiles allows malicious code injection without changing visible manifest files.

### 3.1 Clean Installs Only
* **RULE:** In automated environments, CI pipelines, and verification steps, agents **MUST** use deterministic clean install commands:
  * **npm:** `npm ci --ignore-scripts`
  * **pnpm:** `pnpm install --frozen-lockfile --ignore-scripts`
  * **bun:** `bun install --frozen-lockfile --ignore-scripts`
* Agents **MUST NOT** run naked `npm install` if a lockfile already exists, as it can silently update versions and mutate the lockfile.

### 3.2 Lockfile Inspection & Registry Verification
* All package URLs in lockfiles (`package-lock.json`, `pnpm-lock.yaml`) must point strictly to official, trusted registries (`https://registry.npmjs.org/`).
* Never accept lockfiles with `resolved` links pointing to arbitrary HTTP endpoints, raw GitHub tarballs, or unfamiliar mirrors.

---

## 4. Sandbox & Execution Boundaries

AI agents operate inside containerized or virtualized environments (e.g., Docker Sandboxes `sbx`). The agent must respect the isolation boundaries.

### 4.1 Filesystem Isolation
* The sandbox workspace must be restricted strictly to the project repository.
* Agents must **NEVER** request mounts or read access to host credentials:
  * `~/.ssh`
  * `~/.aws`
  * `~/.npmrc` (host tokens)
  * Shell configuration files (`~/.zshrc`, `~/.bashrc`)
* Any temporary files or logs must remain within the isolated sandbox environment.

### 4.2 Network Egress Rules
* Network access should be limited to necessary endpoints via proxy policies:
  * `registry.npmjs.org`
  * `github.com`
  * Required AI API endpoints
* Agents must never make arbitrary outbound requests to unvetted external IP addresses or unknown domains.

### 4.3 Git & CI Workflow Safeguards
* **RULE:** Agents **MUST NOT** modify or create files inside `.github/workflows/` unless explicitly instructed to edit CI configurations.
* **RULE:** If creating or editing a CI workflow:
  * Never dump environment variables or print secrets to the console.
  * Pin GitHub Actions to full 40-character commit SHAs instead of mutable tags (e.g., `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683` instead of `@v4`).
  * Never introduce curl-pipe-to-bash (`curl ... | sh`) steps in workflows.

---

## 5. Agent Verification Protocol (Before Running Any Command)

Before executing commands that fetch code or build software, the agent must verify:

- [ ] `.npmrc` is present with `ignore-scripts=true` and `allow-git=none`.
- [ ] Dependencies in `package.json` are exact versions (no `^` or `~`).
- [ ] No lifecycle hooks (`preinstall`, `postinstall`) exist in `package.json` scripts without explicit user authorization.
- [ ] Installation is executed with `--ignore-scripts`.
- [ ] Lockfile changes are verified to resolve exclusively to `registry.npmjs.org`.
- [ ] No repository secrets or credentials are being exposed or written to disk.

---

*This guideline is active across all workspaces and agent environments.*
