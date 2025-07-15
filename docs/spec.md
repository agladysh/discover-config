# Config File Discovery Specification

## Purpose
Provide a lightweight, cross-platform, discovery-only mechanism for locating configuration files, directories, and workspace boundaries in a Node.js environment, mimicking standard CLI tool patterns (e.g., Git, SSH, npm) while adhering to XDG Base Directory Specification, Linux Filesystem Hierarchy Standard (FHS), POSIX, and Windows conventions. The system uses a recursive, modular design to return structured data (configs, directories, boundaries) without parsing, prioritizing standards compliance with configurable overrides.

## Scope
- **Discovery only**: Return paths for config files, directories, and boundaries without parsing.
- **Locations**: Search environment, project (`PWD` upwards), user (`$HOME`, `$XDG_CONFIG_HOME`), system (`/etc/`, `$XDG_CONFIG_DIRS`, Windows equivalents), and optional Windows Registry.
- **File patterns**: Support flat (e.g., `.appname.yaml`) and nested (e.g., `.appname/config`) files.
- **Boundaries**: Respect app-specific (e.g., `${APPNAME}_DIR`, `.git`) and system-level (e.g., mountpoints) boundaries, configurable with glob patterns.
- **Standalone discovery**: Export workspace boundary and config directory discovery independently.
- **Return structure**: Use a `pwd-parent[]-workspace-user-system` pentad to represent the filesystem state.
- **Cross-platform**: Support Unix (Linux, macOS) and Windows, including Registry.
- **Standards**: Prioritize XDG, FHS, POSIX, with user-configurable violations.
- **Recursive design**: Consistent logic across scopes, patterns, boundaries, and outputs.

## Design Principles
- **Modularity**: Independent components for configs, directories, boundaries, and output.
- **Recursion**: Apply consistent discovery logic across scopes and components.
- **Simplicity**: Linear precedence, minimal special cases, robust defaults.
- **Standards compliance**: Adhere to XDG, FHS, POSIX unless overridden.
- **Extensibility**: Support future registry-like systems and custom behaviors.
- **Robustness**: Handle permissions, symlinks, long paths, and platform-specific cases.

## Requirements

### 1. Search Locations
Search in a strict, in-order precedence across modular scopes. Each scope applies the same discovery logic for configs, directories, and boundaries.

1. **Environment Variable Override**:
   - Check `${appName.toUpperCase()}_CONFIG` (e.g., `MYAPP_CONFIG`) for a file path.
   - Returns immediately if set (unless disabled via `skipLocations: ['env']`).
   - Example: `MYAPP_CONFIG=/path/to/config.yaml`.

2. **Project Scope**:
   - Traverse from `process.cwd()` to parent directories until a boundary is reached.
   - Discover configs (file patterns), directories (e.g., `.myapp`), and boundaries (e.g., `.git`).
   - Example: `./.myapp.yaml`, `./.myapp/`, `./.git`.

3. **Workspace Scope**:
   - Identify the workspace root (directory containing a boundary like `.git` or `${APPNAME}_DIR`).
   - Discover configs and directories at the workspace root, even if no config exists.
   - Example: `/project/.myapp/config`, `/project/.git`.

4. **User Scope**:
   - Check `$HOME` (`%USERPROFILE%` on Windows) for file patterns and directories.
   - Check `$XDG_CONFIG_HOME` (defaults to `$HOME/.config` or `%APPDATA%`) for `${appname}/`.
   - Example: `~/.myapp.yaml`, `~/.config/myapp/`.

5. **System Scope**:
   - Unix-like:
     - Check `/etc/` for file patterns and directories.
     - Check `$XDG_CONFIG_DIRS` (default: `/etc/xdg`) for `${appname}/`.
   - Windows:
     - Check `%ProgramData%`, `%APPDATA%`, `%LOCALAPPDATA%` for `${appname}/`.
   - Example: `/etc/myapp/config`, `C:\ProgramData\myapp/`.

6. **Windows Registry Scope (Optional)**:
   - Check `HKEY_CURRENT_USER\Software\<appName>\ConfigPath` and `HKEY_LOCAL_MACHINE\Software\<appName>\ConfigPath` for file paths.
   - Disabled by default (enabled via `checkRegistry: true`).
   - Example: `HKEY_CURRENT_USER\Software\MyApp\ConfigPath` → `C:\MyApp\config.yaml`.

**Precedence**:
- Default: `env > project > workspace > user > system > registry`.
- Configurable via `precedence` option (e.g., `['system', 'user', 'project']`).
- Project scope searches from `PWD` to parents until a workspace boundary.

### 2. File Patterns
Apply pattern matching recursively at each scope for configs and directories.
- **Config patterns**:
  - Flat: `${fileName}`, `.${fileName}`, `.${appname}`, `.${appname}.{extension}` (e.g., `config`, `.config`, `.myapp`, `.myapp.yaml`).
  - Nested: `${appname}/${fileName}`, `.${appname}/${fileName}` (e.g., `myapp/config`, `.myapp/config.yaml`).
- **Directory patterns**:
  - Flat: `${appname}`, `.${appname}` (e.g., `myapp/`, `.myapp/`).
  - Nested: `${appname}/`, `.${appname}/` (e.g., `~/.config/myapp/`, `/etc/myapp/`).
- **Extensions**:
  - Default: `['', 'yaml', 'yml', 'json', 'ini']` for configs.
  - Configurable via `extensions` (e.g., `.toml`).
- **Globs**:
  - Support glob patterns for configs and directories (e.g., `*.conf`, `{myapp,.myapp}/*`).
- **Case sensitivity**:
  - Default: Respect filesystem (case-sensitive on Linux, case-insensitive on Windows/macOS).
  - Option: Enforce case-sensitive matching via `caseSensitive: true`.
- **Standards compliance**: Prefer dot-prefixed files (`.myapp`) in `$HOME` per POSIX/XDG unless overridden.

### 3. Boundaries
Apply boundary checks recursively during project scope traversal.
- **Default boundaries**:
  - Filesystem root (e.g., `/`, `C:\`).
  - `${appName.toUpperCase()}_DIR` (e.g., `MYAPP_DIR`) if set.
  - `.git` directory (enabled by default, configurable).
  - Mountpoints (Unix only, detected via `fs.stat.dev`).
- **Configurable boundaries**:
  - Add files, directories, or globs (e.g., `package.json`, `.hg`, `*.root`, `{.git,.svn}`).
  - Disable specific boundaries (e.g., `skipBoundaries: ['.git', 'mountpoints']`).
  - Disable all boundaries (`disableBoundaries: true`).
- **Workspace boundaries**:
  - Default: `.git` (configurable, can be disabled via `disableWorkspaceBoundaries: true`).
  - Support custom markers (e.g., `pnpm-workspace.yaml`).
- **`$GIT_DIR`**:
  - Not a default boundary; can be added via `boundaries: [process.env.GIT_DIR]`.
- **Precedence**:
  - `${APPNAME}_DIR` > `.git` > mountpoints > root.
- **Standards compliance**: Respect FHS (e.g., `/etc/` boundaries) and POSIX traversal unless overridden.

### 4. Symlink Handling
- **Default**: Follow symlinks to real paths (`fs.realpath`).
- **Options**:
  - `as-is`: Do not resolve symlinks.
  - `skip`: Ignore symlinks.
- **Edge cases**:
  - Detect and avoid symlink loops.
  - Respect boundaries for symlinked paths.
- **Standards compliance**: Follow POSIX symlink behavior unless overridden.

### 5. Return Data Structure
Return a recursive `pwd-parent[]-workspace-user-system` pentad structure to represent the filesystem state.
- **Structure**:
  ```typescript
  interface DiscoveryResult {
    pwd: { configs: string[], dirs: string[], boundaries: string[] };
    parents: Array<{ path: string, configs: string[], dirs: string[], boundaries: string[] }>;
    workspace: { path: string | null, configs: string[], dirs: string[], boundaries: string[] };
    user: { configs: string[], dirs: string[] };
    system: { configs: string[], dirs: string[] };
    registry?: { configs: string[] };
  }
  ```
  - `pwd`: Configs, directories, and boundaries in `process.cwd()`.
  - `parents`: Array of parent directories (up to workspace boundary), each with configs, dirs, and boundaries.
  - `workspace`: Workspace root (e.g., directory with `.git` or `${APPNAME}_DIR`), with configs, dirs, and boundaries; `null` if no boundary found.
  - `user`: Configs and directories in `$HOME`, `$XDG_CONFIG_HOME`.
  - `system`: Configs and directories in `/etc/`, `$XDG_CONFIG_DIRS`, Windows equivalents.
  - `registry`: Config paths from Windows Registry (if enabled).
- **Selection**:
  - Default: Return first config path in precedence order (`env > project > workspace > user > system > registry`).
  - Configurable via `select` option (e.g., `['configs', 'dirs', 'boundaries']`) to return specific components.
  - Example: `select: ['configs']` returns only config paths; `select: ['dirs', 'boundaries']` returns directories and boundaries.
- **Validation**:
  - Configs: Regular files only (not directories or unresolved symlinks).
  - Dirs: Existing directories matching patterns.
  - Boundaries: Files or directories matching boundary globs.
  - Skip inaccessible paths.

### 6. Standalone Discovery
- **Workspace Boundary Discovery**:
  - Function: `findWorkspaceBoundary(appName: string, options?: Options): Promise<string | null>`.
  - Returns: Path to workspace root (e.g., directory with `.git` or `${APPNAME}_DIR`) or `null`.
  - Uses same boundary logic as main discovery.
- **App Config Dir Discovery**:
  - Function: `findAppConfigDirs(appName: string, options?: Options): Promise<string[]>`.
  - Returns: Array of app-specific directories (e.g., `./.myapp/`, `~/.config/myapp/`).
  - Uses same directory patterns as main discovery.
- **Integration**: Both functions can be used standalone or within main discovery (`findAppConfig`).

### 7. Windows Support
- **Path handling**:
  - Use `path.normalize` for cross-platform paths.
  - Handle case-insensitive filesystems.
- Support long paths (>260 chars) with `\\?\` prefix.
  ```javascript
  // Normalize long paths on Windows
  function withLongPathPrefix(p) {
    if (process.platform === 'win32' && p.length >= 260 && !p.startsWith('\\\\?\\')) {
      return '\\?\\' + path.resolve(p);
    }
    return p;
  }
  ```
- **Mappings**:
  - `$HOME`: `%USERPROFILE%`.
  - `$XDG_CONFIG_HOME`: `%APPDATA%` or `%LOCALAPPDATA%`.
  - `/etc/`: `%ProgramData%`, `%APPDATA%`, `%LOCALAPPDATA%`.
- **Registry**:
- Optional check for `HKEY_CURRENT_USER\Software\<appName>\ConfigPath`, `HKEY_LOCAL_MACHINE\Software\<appName>\ConfigPath`.
- Skip invalid/missing keys.
  ```javascript
  // Read config path from Windows Registry if module is present
  let WinReg;
  try {
    WinReg = require('winreg');
  } catch {
    WinReg = null;
  }

  async function readRegistryPath(appName) {
    if (!WinReg) return null; // module not installed
    const reg = new WinReg({ hive: WinReg.HKCU, key: `\\Software\\${appName}` });
    return new Promise(resolve =>
      reg.get('ConfigPath', (err, item) => resolve(err ? null : item.value))
    );
  }
  ```
- **Edge cases**:
  - Handle case variations (e.g., `.MyApp` vs `.myapp`).
  - Skip inaccessible paths.

### 8. Configurability
- **Locations**:
  - Enable/disable scopes (e.g., `skipLocations: ['/etc', 'registry']`).
  - Add custom paths (e.g., `/opt/appname`).
- **File patterns**:
  - Custom `fileName` (default: `config`).
  - Custom `extensions` (default: `['', 'yaml', 'yml', 'json', 'ini']`).
  - Glob patterns (e.g., `*.conf`).
- **Boundaries**:
  - Custom files/dirs/globs (default: `.git`).
  - Disable boundaries (e.g., `disableBoundaries: true`).
- **Selection**:
  - `select`: Choose components (e.g., `['configs', 'dirs', 'boundaries']`).
- **Other**:
  - `symlinkBehavior`: `follow`, `as-is`, `skip` (default: `follow`).
  - `caseSensitive`: Enforce case sensitivity.
  - `precedence`: Custom order (default: `['env', 'project', 'workspace', 'user', 'system', 'registry']`).
  - `checkRegistry`: Enable Registry checks (default: false).
- **Standards overrides**:
  - Allow violations of XDG/FHS/POSIX (e.g., custom paths, non-dot-prefixed files) via options.

### 9. Error Handling
- Skip inaccessible paths without errors.
- Handle malformed env vars or Registry keys.
- Fallbacks: `$XDG_CONFIG_HOME` → `$HOME/.config`, `$XDG_CONFIG_DIRS` → `/etc/xdg`.
- Avoid symlink loops.
- Handle deep directory trees and long paths.

### 10. Performance
- Minimize filesystem calls (e.g., cache `fs.stat`).
  ```javascript
  // Simple in-memory cache for fs.stat results
  const statCache = new Map();

  async function statCached(p) {
    if (statCache.has(p)) return statCache.get(p);
    try {
      const s = await fs.promises.stat(p);
      statCache.set(p, s);
      return s;
    } catch {
      statCache.set(p, null);
      return null;
    }
  }
  ```
- Parallelize independent scope checks.
- Handle large directory trees and pattern sets efficiently.

### 11. Constraints
- **Dependencies**: Node.js built-ins (`fs`, `path`, `os`); optional Registry module (e.g., `node-winreg`).
- **Node.js**: Support >= 14 (LTS as of 2025).
- **No parsing**: Return paths only.
- **No CLI**: Process env vars only for overrides (`$APPNAME_CONFIG`, `$APPNAME_DIR`).

### 12. Notes
- **Standards precedence**:
  - Prioritize XDG (e.g., `$XDG_CONFIG_HOME`, `$XDG_CONFIG_DIRS`), FHS (e.g., `/etc/`), and POSIX (e.g., dot-prefixed files, symlink resolution) unless overridden.
  - Allow reasonable violations via options (e.g., `searchLocations`, `patterns`, `skipLocations`) per KISS principles.
- **Future work**:
  - Support registry-like mechanisms for macOS (e.g., `~/Library/Preferences`, `plist` files) and Linux (e.g., `/usr/share`, `dconf`) for config file paths.
  - YAGNI: Defer implementation, but ensure API extensibility (e.g., new scope types).
- **Standalone discovery**:
  - Workspace boundary and config dir discovery are exposed as standalone functions for flexibility.
  - These can be used independently or combined with config file discovery.

## Recursive Design
- **Scopes**: Each scope (env, project, workspace, user, system, registry) uses a single discovery function:
  - Input: Scope root, patterns (configs/dirs), boundaries.
  - Output: Structured pentad (configs, dirs, boundaries).
- **File/directory patterns**: Recursively apply flat, nested, and glob patterns.
- **Boundaries**: Recursively check during project traversal.
- **Return structure**: Recursive pentad (`pwd-parent[]-workspace-user-system`) for flexible querying.
- **Default config**:
  - Precedence: `env > project > workspace > user > system > registry`.
  - Patterns: `['config', '.config', '.${appname}', '.${appname}.{yaml,yml,json,ini}', '${appname}/config', '.${appname}/config']` for configs; `['${appname}', '.${appname}']` for dirs.
  - Boundaries: `${APPNAME}_DIR`, `.git`, mountpoints, root.
  - Symlinks: Follow.
  - Registry: Disabled.
  - Select: `['configs']` (first config path).

## Example Scenarios
1. **Default (configs only)**:
   - App: `myapp`, select: `['configs']`.
   - Returns: `./.myapp/config`.
2. **Workspace boundary**:
   - `findWorkspaceBoundary('myapp')`.
   - Returns: `/project/.git`.
3. **Full pentad**:
   - Select: `['configs', 'dirs', 'boundaries']`.
   - Returns:
     ```json
     {
       pwd: { configs: [], dirs: ['./.myapp'], boundaries: [] },
       parents: [{ path: '../', configs: [], dirs: [], boundaries: [] }],
       workspace: { path: '/project', configs: ['/project/.myapp.yaml'], dirs: ['/project/.myapp'], boundaries: ['/project/.git'] },
       user: { configs: ['~/.config/myapp/config'], dirs: ['~/.config/myapp'] },
       system: { configs: ['/etc/myapp/config'], dirs: ['/etc/myapp'] },
       registry: { configs: [] }
     }
     ```
4. **Windows Registry**:
   - `checkRegistry: true`, select: `['configs']`.
   - Returns: `C:\MyApp\config.yaml`.

## API Sketch
```javascript
interface Options {
  fileName?: string; // Default: 'config'
  extensions?: string[]; // Default: ['', 'yaml', 'yml', 'json', 'ini']
  patterns?: string[]; // Globs for configs/dirs (e.g., ['*.conf', '{myapp,.myapp}'])
  envOverride?: string; // Default: `${appName.toUpperCase()}_CONFIG`
  envDirOverride?: string; // Default: `${appName.toUpperCase()}_DIR`
  searchLocations?: string[]; // Custom paths
  skipLocations?: string[]; // Skip paths (e.g., ['/etc', 'registry'])
  boundaries?: string[]; // Files/dirs/globs (default: ['.git'])
  skipBoundaries?: string[]; // Skip specific boundaries
  disableBoundaries?: boolean;
  caseSensitive?: boolean;
  symlinkBehavior?: 'follow' | 'as-is' | 'skip'; // Default: 'follow'
  select?: string[]; // Default: ['configs'] (options: 'configs', 'dirs', 'boundaries')
  precedence?: string[]; // Default: ['env', 'project', 'workspace', 'user', 'system', 'registry']
  checkRegistry?: boolean; // Default: false
}

async function findAppConfig(appName: string, options?: Options): Promise<string | DiscoveryResult | null>;
async function findWorkspaceBoundary(appName: string, options?: Options): Promise<string | null>;
async function findAppConfigDirs(appName: string, options?: Options): Promise<string[]>;
```

## Notes
- **Standards precedence**:
  - Prioritize XDG (e.g., `$XDG_CONFIG_HOME`, `$XDG_CONFIG_DIRS`), FHS (e.g., `/etc/`), POSIX (e.g., dot-prefixed files, symlink resolution) unless overridden.
  - Allow violations via options (e.g., `searchLocations`, `patterns`, `skipLocations`) per KISS.
- **Future work**:
  - Support macOS (e.g., `~/Library/Preferences`, `plist`) and Linux (e.g., `/usr/share`, `dconf`) registry-like mechanisms for config paths.
  - YAGNI: Defer implementation, but ensure API extensibility.
- **Standalone discovery**:
  - `findWorkspaceBoundary` and `findAppConfigDirs` are standalone for flexibility.
  - Integrated with `findAppConfig` via `select` option for unified output.
