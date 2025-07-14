# Config File Discovery Specification

## Purpose
Provide a lightweight Node.js library for locating application configuration files and directories. The library discovers paths only and performs no parsing. It works on Linux, macOS and Windows and adheres to the XDG Base Directory Specification and common conventions.

## Quick Start
```javascript
import { findAppConfig } from 'discover-config';

const path = await findAppConfig('myapp');
// './.myapp/config.yaml'
```
Use `findWorkspaceBoundary()` to locate the project root and `findAppConfigDirs()` to list discovered config directories.

## Search Order
Scopes are checked in the following order until a config file is found. The order can be changed via the `precedence` option and individual scopes may be skipped.
1. **Environment** – `${APPNAME}_CONFIG` or `${APPNAME}_DIR` variables.
2. **Project** – current directory upward to the first workspace boundary.
3. **Workspace** – the directory containing a boundary such as `.git` or `${APPNAME}_DIR`.
4. **User** – `$XDG_CONFIG_HOME` (fallback `~/.config`) and `$HOME`.
5. **System** – `/etc` and `$XDG_CONFIG_DIRS` (default `/etc/xdg`), plus Windows equivalents.
6. **Registry** – optional Windows registry keys if `checkRegistry` is enabled.

Default precedence is `env > project > workspace > user > system > registry`.

## File Patterns
At each scope the library looks for config files and directories using the following defaults:
- **Config files** – `config`, `.config`, `.${appName}`, `.${appName}.{ext}` and `${appName}/config`.
- **Config directories** – `${appName}` or `.${appName}`; `${appName}/` within XDG directories.
- **Extensions** – `['', 'yaml', 'yml', 'json', 'ini']`.
Patterns and extensions may be customised through options.

## Workspace Boundaries
When traversing parent directories a boundary stops the search. By default a boundary is detected if:
- the `${APPNAME}_DIR` environment variable points to the directory,
- a `.git` directory is present,
- the path is a filesystem root or mount point.
Additional files or globs can be listed via the `boundaries` option.

## Symlink Handling
Paths are resolved with `fs.realpath` by default. Set `symlinkBehavior` to `as-is` to keep symlinks or `skip` to ignore them. Symlink loops are detected and skipped.

## Return Structure
`findAppConfig()` returns either the first config path or a detailed structure:
```typescript
interface ScopeResult {
  configFiles: string[];
  configDirs: string[];
  boundaries: string[];
}
interface ParentResult extends ScopeResult {
  path: string;
}
interface DiscoveryResult {
  pwd: ScopeResult;
  parents: ParentResult[];
  workspace: ScopeResult & { path: string | null };
  user: ScopeResult;
  system: ScopeResult;
  registry?: ScopeResult;
}
```
Use the `select` option to choose which pieces to return.

## API
```typescript
interface Options {
  fileName?: string;
  extensions?: string[];
  patterns?: string[];
  envOverride?: string;
  envDirOverride?: string;
  searchLocations?: string[];
  skipLocations?: string[];
  boundaries?: string[];
  skipBoundaries?: string[];
  disableBoundaries?: boolean;
  caseSensitive?: boolean;
  symlinkBehavior?: 'follow' | 'as-is' | 'skip';
  select?: Array<'configFiles' | 'configDirs' | 'boundaries'>;
  precedence?: string[];
  checkRegistry?: boolean;
}

async function findAppConfig(appName: string, options?: Options): Promise<string | DiscoveryResult | null>;
async function findWorkspaceBoundary(appName: string, options?: Options): Promise<string | null>;
async function findAppConfigDirs(appName: string, options?: Options): Promise<string[]>;
```

## Default Options
| Option | Default | Example |
| ------ | ------- | ------- |
| `fileName` | `'config'` | `'settings'` |
| `extensions` | `['', 'yaml', 'yml', 'json', 'ini']` | `['toml']` |
| `precedence` | `['env','project','workspace','user','system','registry']` | `['env','user']` |
| `symlinkBehavior` | `'follow'` | `'as-is'` |
| `checkRegistry` | `false` | `true` |

## Implementation Notes
- Works with Node.js ≥14 using only built-in modules. Registry access requires an optional Windows package.
- Inaccessible paths are skipped silently.
- Designed for extension through additional scope providers.

## References
- [XDG Base Directory Specification](https://specifications.freedesktop.org/basedir-spec/latest/)
- [Filesystem Hierarchy Standard](https://refspecs.linuxfoundation.org/FHS_3.0/fhs-3.0.html)
- [POSIX](https://pubs.opengroup.org/onlinepubs/9699919799/)
