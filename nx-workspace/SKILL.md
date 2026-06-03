---
name: nx-workspace
description: Expert knowledge for NX monorepo management — commands, generators, executors, project graph, caching, and library organization. Use when working with NX workspaces, running affected builds, creating libraries, configuring tags, or optimizing CI pipelines.
triggers:
  - nx workspace
  - nx monorepo
  - nx generate
  - nx affected
  - nx library
  - project graph
  - nx cache
  - nx executor
  - nx generator
  - nx tags
---

# NX Workspace Expert

You are now loaded with deep knowledge about NX monorepo management. Apply these patterns when helping the user work with their NX workspace.

## Core Commands

### Running Tasks
```bash
nx run <project>:<target>              # run a target
nx run <project>:<target>:<config>     # with configuration
nx run-many -t build test lint         # run targets across all projects
nx run-many -t build --projects=app1,lib1  # specific projects
nx affected -t build test lint         # only affected projects
nx affected -t test --base=main --head=HEAD
```

### Project Graph
```bash
nx graph                               # open interactive graph in browser
nx graph --focus=<project>             # focus on one project
nx show projects                       # list all projects
nx show project <name>                 # show project details
```

### Code Generation
```bash
nx generate @nx/angular:library my-lib --directory=libs/shared
nx generate @nx/angular:component my-comp --project=my-lib
nx list                                # list available plugins
nx list @nx/angular                    # list generators for a plugin
```

### Workspace Info
```bash
nx reset                               # clear cache
nx daemon --start / --stop            # NX daemon management
nx report                              # workspace plugin versions
```

## Library Organization (Recommended Structure)

```
libs/
  <domain>/
    feature-<name>/     # smart components, pages, routing
    data-access/        # services, state, API calls, NgRx
    ui/                 # dumb/presentational components
    util/               # pure functions, pipes, guards, models
```

### Library Types and Rules
| Type | Contains | Can depend on |
|---|---|---|
| `feature` | Smart components, containers, routes | data-access, ui, util |
| `data-access` | Services, NgRx, HTTP | util |
| `ui` | Dumb components, directives | util |
| `util` | Pure functions, models, pipes | (nothing) |

## Tags and Constraints

Define tags in `project.json`:
```json
{
  "tags": ["scope:shared", "type:ui"]
}
```

Enforce constraints in `nx.json`:
```json
{
  "targetDefaults": {},
  "namedInputs": {},
  "plugins": ["@nx/eslint/plugin"],
  "generators": {},
  "targetDependencies": {}
}
```

Use `@nx/eslint-plugin` `enforce-module-boundaries` rule:
```json
{
  "rules": {
    "@nx/enforce-module-boundaries": ["error", {
      "depConstraints": [
        { "sourceTag": "type:feature", "onlyDependOnLibsWithTags": ["type:data-access", "type:ui", "type:util"] },
        { "sourceTag": "type:data-access", "onlyDependOnLibsWithTags": ["type:util"] },
        { "sourceTag": "type:ui", "onlyDependOnLibsWithTags": ["type:util"] },
        { "sourceTag": "scope:shared", "onlyDependOnLibsWithTags": ["scope:shared"] }
      ]
    }]
  }
}
```

## Caching

### Local Cache
NX caches task outputs locally in `.nx/cache`. Tasks are cached by inputs hash.

Configure inputs in `nx.json`:
```json
{
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": ["default", "!{projectRoot}/**/*.spec.ts", "!{projectRoot}/tsconfig.spec.json"]
  },
  "targetDefaults": {
    "build": {
      "inputs": ["production", "^production"],
      "outputs": ["{projectRoot}/dist"],
      "cache": true
    },
    "test": {
      "inputs": ["default", "^production"],
      "cache": true
    }
  }
}
```

### Remote Cache (NX Cloud)
```bash
nx connect                             # connect to NX Cloud
```
- Shares cache across CI runs and team members
- First run computes, subsequent runs replay from cache
- NX Cloud also provides distributed task execution (DTE)

## Executors (Custom Build Targets)

Define in `project.json`:
```json
{
  "targets": {
    "build": {
      "executor": "@angular-devkit/build-angular:browser",
      "options": { "outputPath": "dist/my-app" },
      "configurations": {
        "production": { "optimization": true },
        "development": { "optimization": false }
      }
    }
  }
}
```

Custom executor structure:
```
tools/executors/my-executor/
  executor.ts
  schema.json
  schema.d.ts
```

## Generators (Code Scaffolding)

Custom generator structure:
```
tools/generators/my-generator/
  generator.ts
  schema.json
  schema.d.ts
```

Generator template:
```typescript
import { Tree, formatFiles, generateFiles } from '@nx/devkit';
export async function myGenerator(tree: Tree, options: MyGeneratorSchema) {
  generateFiles(tree, path.join(__dirname, 'files'), options.projectRoot, options);
  await formatFiles(tree);
}
```

## Affected Commands (CI Optimization)

```bash
# In CI: compare against base branch
nx affected -t build --base=origin/main
nx affected -t test --base=origin/main --parallel=3
nx affected --graph                    # visualize affected subgraph
```

Affected is calculated from:
1. Changed files since base
2. Project dependency graph (if a lib changes, all dependents are affected)

## `nx.json` Key Settings

```json
{
  "defaultBase": "main",
  "workspaceLayout": {
    "appsDir": "apps",
    "libsDir": "libs"
  },
  "parallel": 3,
  "cacheDirectory": ".nx/cache",
  "useDaemonProcess": true,
  "plugins": ["@nx/angular/plugin"]
}
```

## CI Pipeline Best Practices

```yaml
# GitHub Actions example
- name: NX Affected Build
  run: npx nx affected -t build --base=origin/main --head=HEAD --parallel=3

- name: NX Affected Test
  run: npx nx affected -t test --base=origin/main --parallel=2 --ci

- name: NX Affected Lint
  run: npx nx affected -t lint --base=origin/main
```

- Always use `--base=origin/main` not `--base=main` in CI (fetch remote refs)
- Use `--parallel` to speed up task execution
- Cache `.nx/cache` directory between CI runs

## Checklist
- [ ] Libraries organized by domain and type (feature/data-access/ui/util)?
- [ ] Tags defined on all projects?
- [ ] Module boundary constraints configured in ESLint?
- [ ] `namedInputs` defined to maximize cache hits?
- [ ] `affected` used in CI (not `run-many`)?
- [ ] NX daemon enabled for local development?
- [ ] Remote cache configured for team cache sharing?
