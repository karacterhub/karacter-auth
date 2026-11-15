>Repository Instructions for GitHub Copilot: Karacterhub Auth

Role & Mission 

You are an expert TypeScript library engineer working inside the GitHub organization @karacterhub. Your mission is to scaffold, implement, test, and prepare for publish a production-grade npm package named @karacter/auth — a unified toolkit that bundles three concerns for our internal Next.js projects: 

1. Authentication (a thin, opinionated wrapper over Auth.js / next-auth v5)
2. MCP connection system (client + server helpers using @modelcontextprotocol/sdk)
3. OAuth integration system (provider-agnostic OAuth via arctic) 

The package must be installable via npm i @karacter/auth, bun add @karacter/auth, or pnpm add @karacter/auth, and consumable through subpath exports (@karacter/auth/auth, @karacter/auth/mcp, @karacter/auth/oauth). 

Work in the repository karacterhub/karacter-auth. If the repo does not exist, create it under the org with: 

· Visibility: private by default (we will flip to public before first npm publish, or keep private with GitHub Packages — see Publishing section)
· License: MIT
· Initial files: .gitignore (Node), README.md, LICENSE
· Default branch: main 

─── 

Working Protocol (read before touching anything) 

1. Branch strategy: Create a branch feat/scaffold-package off main. Do all work there. Open a draft PR early (after the initial package.json + directory scaffold) so I can review direction.
2. Commits: Use Conventional Commits (feat:, chore:, test:, docs:, fix:). One logical change per commit.
3. Stop-and-ask checkpoints — do not proceed past these without confirmation:
· Before adding any dependency not explicitly listed in this prompt.
· Before changing the exports map shape from what is specified below.
· Before running npm publish, git tag, or any command that pushes to a remote registry.
· Before making the GitHub repo public.
4. Never commit secrets, .env files, node_modules/, dist/, or npm tokens.
5. Never write "placeholder" tests that trivially pass. If a test can't be written yet, leave a // TODO(test): comment and add it to the PR checklist.
6. TypeScript discipline: strict: true, no any without an inline // eslint-disable and justification comment, no @ts-ignore.
7. Report format: At the end of your work session, respond with a checklist covering every item in the Deliverables section (done / not done + reason). 

─── 

Technology Constraints 

Concern Requirement
Runtime Node >= 18. Must work in Bun and Next.js (Node runtime and edge where feasible)
Language TypeScript 5.4+, strict: true
Module system ESM-first ("type": "module"), ship dual ESM + CJS via tsup
Build tool tsup
Test framework vitest
Package manager npm for the library repo (do not mix lockfiles) 

Dependencies you may add (and only these): 

Runtime dependencies: 

· @modelcontextprotocol/sdk (latest stable)
· arctic (^3)
· zod (^3) 

peerDependencies: 

· next (>= 14) — optional
· next-auth (^5.0.0-beta) — optional
· react (>= 18) — optional 

devDependencies (add freely as needed): 

· tsup, typescript, vitest, @types/node, @types/react, next, next-auth, react, react-dom, publint, @arethetypeswrong/cli 

Mark next, next-auth, and react as optional: true in peerDependenciesMeta so consumers who only use the MCP or OAuth subpaths don't need to install them. 

─── 

Repository Layout (target) 

karacter-auth/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── publish.yml
├── .gitignore
├── .npmrc
├── LICENSE
├── README.md
├── package.json
├── tsconfig.json
├── tsup.config.ts
├── vitest.config.ts
├── src/
│   ├── index.ts
│   ├── auth/
│   │   ├── index.ts
│   │   ├── config.ts
│   │   ├── session.ts
│   │   └── config.test.ts
│   ├── mcp/
│   │   ├── index.ts
│   │   ├── client.ts
│   │   ├── server.ts
│   │   └── client.test.ts
│   ├── oauth/
│   │   ├── index.ts
│   │   ├── providers.ts
│   │   └── providers.test.ts
│   └── cli/
│       └── init.ts
├── examples/
│   └── nextjs-app/
│       ├── README.md
│       └── app/api/auth/[...nextauth]/route.ts
└── dist/           (gitignored, generated)


─── 

Module Specifications 

A. @karacter/auth/auth — Auth.js wrapper 

File: src/auth/config.ts 

Implement and export createKaracterAuth(options): 

· Extends NextAuthConfig.
· Adds an optional verifyUser?: (input: { email: string; password: string }) => Promise<{ id: string; email: string; name?: string } | null>.
· Registers a default Credentials provider whose authorize validates input with a zod schema (email: string.email(), password: string.min(8)), calls verifyUser, and returns the user or null.
· Merges any providers and callbacks passed in by the caller after the defaults (caller overrides win).
· Default session strategy: jwt.
· Returns { handlers, auth, signIn, signOut, config }. 

File: src/auth/session.ts 

· Re-export getServerSession from next-auth.
· Provide a small helper requireSession(auth) that throws a typed KaracterAuthError if no session — this is what most of our Next.js apps want. 

File: src/auth/index.ts 

· Barrel re-export. 

Acceptance criteria: 

· Calling createKaracterAuth() with no options returns working Auth.js primitives.
· verifyUser returning null causes authorize to return null.
· Invalid email/password shape short-circuits to null without calling verifyUser.
· Unit tests cover: happy path, invalid email, short password, verifyUser rejects, caller-supplied provider is preserved. 

B. @karacter/auth/mcp — MCP client & server helpers 

File: src/mcp/client.ts 

Export createMcpClient(opts) with: 

ts
export interface McpClientOptions {
  name: string
  version?: string
  command: string
  args?: string[]
  env?: Record<string, string>
}


· Instantiates a StdioClientTransport and a Client from @modelcontextprotocol/sdk, connects, and returns the connected client.
· Wrap connection errors in a KaracterMcpError with a clear message (e.g. Failed to connect MCP client "$`{name}"). 

File: src/mcp/server.ts 

Export minimal helpers to define MCP tools with zod schemas and register them on an MCP Server. Keep this module thin — do not reimplement the SDK, just provide org-standard ergonomics: 

· defineTool({ name, description, schema, handler }) returning the object shape the SDK expects.
· createKaracterMcpServer({ name, version, tools }) that wires them up. 

Acceptance criteria: 

· createMcpClient unit-tested with a mocked transport (do not spawn real processes in tests).
· defineTool returns a valid MCP tool descriptor.
· Types exported from subpath are usable without importing the SDK directly. 

C. @karacter/auth/oauth — OAuth integration 

File: src/oauth/providers.ts 

· Export a providers object with at least github, google, discord, each a factory (config: OAuthConfig) => ArcticProvider.
· Export OAuthConfig type: { clientId, clientSecret, redirectURI }.
· Re-export generateState, generateCodeVerifier, and the relevant arctic provider classes from src/oauth/index.ts. 

Acceptance criteria: 

· Each factory returns an arctic instance configured with the provided values.
· A unit test asserts the returned instance is an instance of the correct arctic class. 

D. @karacter/auth (root barrel) 

src/index.ts re-exports everything from ./auth, ./mcp, ./oauth. Root import must not eagerly import next-auth at module load if the consumer only wants MCP — use the subpath entries as the primary consumer path and keep the root barrel as a convenience. 

E. CLI — npx @karacter/auth init 

File: src/cli/init.ts 

· Shebang #!/usr/bin/env node.
· Generates karacter-auth.config.ts in process.cwd() if it doesn't already exist.
· Does not overwrite. Prints a clear message either way.
· Wire "bin": { "karacter-auth": "./dist/cli/init.js" } in package.json. 

─── 

package.json Requirements 

jsonc
{
  "name": "@karacter/auth",
  "version": "0.1.0",
  "description": "Karacter auth, MCP, and OAuth toolkit",
  "type": "module",
  "sideEffects": false,
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".":        { "types": "./dist/index.d.ts",       "import": "./dist/index.js",       "require": "./dist/index.cjs" },
    "./auth":   { "types": "./dist/auth/index.d.ts",  "import": "./dist/auth/index.js",  "require": "./dist/auth/index.cjs" },
    "./mcp":    { "types": "./dist/mcp/index.d.ts",   "import": "./dist/mcp/index.js",   "require": "./dist/mcp/index.cjs" },
    "./oauth":  { "types": "./dist/oauth/index.d.ts", "import": "./dist/oauth/index.js", "require": "./dist/oauth/index.cjs" },
    "./package.json": "./package.json"
  },
  "files": ["dist", "README.md", "LICENSE"],
  "bin": { "karacter-auth": "./dist/cli/init.js" },
  "scripts": {
    "build": "tsup",
    "dev": "tsup --watch",
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc --noEmit",
    "lint:pkg": "publint && attw --pack .",
    "prepublishOnly": "npm run build && npm run test && npm run typecheck"
  },
  "engines": { "node": ">=18" },
  "publishConfig": { "access": "public" },
  "repository": { "type": "git", "url": "git+https://github.com/karacterhub/karacter-auth.git" },
  "homepage": "https://github.com/karacterhub/karacter-auth#readme",
  "bugs": { "url": "https://github.com/karacterhub/karacter-auth/issues" },
  "license": "MIT"
}


Add the dependency blocks per the table above. Do not deviate from the exports map shape without asking. 

─── 

tsup.config.ts 

ts
import { defineConfig } from 'tsup' 

export default defineConfig({
  entry: {
    index:          'src/index.ts',
    'auth/index':   'src/auth/index.ts',
    'mcp/index':    'src/mcp/index.ts',
    'oauth/index':  'src/oauth/index.ts',
    'cli/init':     'src/cli/init.ts',
  },
  format: ['esm', 'cjs'],
  dts: true,
  splitting: true,
  sourcemap: true,
  clean: true,
  external: ['next', 'next-auth', 'react', 'react-dom'],
})


Verify the built CLI still begins with #!/usr/bin/env node after build. If tsup strips it, add banner: { js: '#!/usr/bin/env node' } on the cli/init entry only. 

─── 

tsconfig.json 

json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "lib": ["ES2022", "DOM"],
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "jsx": "react-jsx",
    "outDir": "dist",
    "rootDir": "src"
  },
  "include": ["src"],
  "exclude": ["dist", "node_modules", "examples"]
}


─── 

Testing 

· vitest with one test file per module listed above.
· Mock next-auth and @modelcontextprotocol/sdk boundaries — do not require a live Next.js app or an external MCP server to run tests.
· Coverage target: ≥ 80% line coverage on src/.
· Add vitest.config.ts with a coverage block. 

─── 

CI & Publishing 

.github/workflows/ci.yml 

Trigger: push to any branch, pull_request to main. 

Steps: 

1. actions/checkout@v4
2. actions/setup-node@v4 with node-version: 20 and cache: npm
3. npm ci
4. npm run typecheck
5. npm run build
6. npm run test
7. npm run lint:pkg 

.github/workflows/publish.yml 

Trigger: push of tags matching v*. 

Steps: 

1. Checkout, setup-node with registry-url: https://registry.npmjs.org
2. npm ci && npm run build && npm run test
3. npm publish --access public
· env: NODE_AUTH_TOKEN: `${{ secrets.NPM_TOKEN }} 

Do not add the NPM_TOKEN secret yourself. Instead, note it in the PR description as a manual step for me. 

.npmrc 

//registry.npmjs.org/:_authToken=$`{NPM_TOKEN}


Leave it out of git only if the token leaks in — it does not; the variable reference is safe to commit. 

Private-installation fallback 

If we decide to keep the package private, add to README.md a clearly-marked section showing consumers how to configure: 

@karacter:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=`${GITHUB_TOKEN}


Do not implement this proactively — just leave the note. 

─── 

README Requirements 

Include: 

· One-paragraph description.
· Install commands for npm, bun, pnpm.
· A "Quick start" for each of the three concerns (auth, mcp, oauth) with a minimal, correct code snippet.
· The npx @karacter/auth init usage.
· Peer-dependency note: consumers must install next-auth themselves.
· A "Consuming from another org repo" section covering the local npm link workflow. 

─── 

Local Dev / Consumption Test (do this before declaring done) 

1. npm run build — verify dist/ contains index., auth/index., mcp/index., oauth/index., cli/init.* and all .d.ts files.
2. npx publint — must pass.
3. npx @arethetypeswrong/cli --pack . — no red flags on our entry points.
4. In a scratch Next.js 14 app outside this repo (create a temporary one under /tmp — do not commit it):
· npm link ../path/to/karacter-auth
· npm link @karacter/auth
· Import from @karacter/auth/auth and call createKaracterAuth(). Confirm it returns handlers, auth, signIn, signOut.
· Import from @karacter/auth/mcp and @karacter/auth/oauth — confirm types resolve.
5. Document the result of step 4 in the PR description. 

─── 

Guardrails — Do Not 

· Do not add dependencies not listed above.
· Do not publish to npm, tag a release, or push a git tag.
· Do not make the repo public.
· Do not create or modify files in any other @karacterhub repository.
· Do not add a postinstall, prepare, or any lifecycle script that runs network access.
· Do not use any, @ts-ignore, or @ts-expect-error without an inline justification.
· Do not write tests that only assert expect(true).toBe(true).
· Do not collapse the subpath structure into a single entry point.
· Do not modify .github/CODEOWNERS (if present) or branch protection settings. 

─── 

Definition of Done 

The task is complete when all of these are true: 

☐ Repo karacterhub/karacter-auth exists with main and feat/scaffold-package branches.
☐ Draft PR opened from feat/scaffold-package → main with a checklist mapping to this prompt.
☐ All files from the target layout exist.
☐ npm run typecheck, npm run test, npm run build, npm run lint:pkg all pass locally.
☐ CI workflow passes on the PR.
☐ dist/ builds cleanly with dual ESM/CJS + .d.ts for every entry.
☐ publint and attw are green.
☐ Local consumption test (above) succeeds and is documented in the PR.
☐ README covers all three concerns + npx init + peer-dep note.
☐ PR description lists the manual steps I must perform: create NPM_TOKEN secret, decide repo visibility, approve first publish. 

─── 

Deliverables — Report Back With 

At the end of your work session, reply with: 

1. Repo URL and PR URL.
2. A checklist against Definition of Done above with ✅ / ❌ + one-line reason.
3. A list of any deviations from this prompt (dependency added, file renamed, spec reinterpreted) with justification.
4. A list of manual steps left for me.
5. Suggested next steps after merge (e.g. "tag v0.1.0, publish, then bump consumers").
