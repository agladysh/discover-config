# Review Notes on `docs/spec.md`

## Overview

This document captures my initial understanding, questions, and concerns after reading the "Config File Discovery Specification". The spec describes a Node.js library for discovering configuration files, configuration directories, and workspace boundaries in a predictable, standards‑aligned way across Unix and Windows platforms. The implementation is expected to be recursive, modular, cross‑platform and respect conventions like XDG, FHS, POSIX, and (optionally) the Windows Registry. No CLI is provided; only library functions that return filesystem paths without parsing.

## General Impressions

* The spec is detailed and covers many edge cases. It emphasizes recursion and a consistent design across scopes (env, project, workspace, user, system, registry).
* A "pentad" data structure is proposed for returning results: `pwd`, `parents`, `workspace`, `user`, `system`, plus optional `registry`. This is flexible but will require careful construction.
* The API sketch shows asynchronous functions returning either a single string (when only the first config path is requested) or a full `DiscoveryResult` object. This conditional return type could be tricky for TypeScript consumers, but it's workable in JS.
* Dependencies are restricted to Node built‑ins, meaning glob matching, symlink handling, and Registry access must either use built‑ins or minimal optional modules (e.g., `node-winreg`).
* The spec references many standards (XDG, FHS, POSIX) and expects the implementation to follow them by default while allowing overrides.

## Questions & Clarifications Needed

1. **Environment variable override path handling**
   * If `${APPNAME}_CONFIG` points to a relative path, do we resolve it relative to `process.cwd()` or treat it as already absolute? Should `~` be expanded?
   * Should we verify the file exists before returning, or simply return the path? The spec implies we should skip inaccessible paths, so existence check seems required.
2. **Precedence customization**
   * When the `precedence` option is provided, should invalid entries be ignored or cause an error? What if the list omits some scopes entirely?
   * Are custom scopes allowed via `precedence`, or must the values come from the predefined set (`env`, `project`, `workspace`, `user`, `system`, `registry`)?
3. **Project scope traversal**
   * The spec states we traverse from `process.cwd()` upward until a boundary is reached. When multiple boundaries exist (e.g., nested Git repos), do we stop at the first one encountered or continue to the filesystem root collecting all boundaries?
   * How should `skipBoundaries` and `disableBoundaries` interact with this traversal? For example, if `.git` is skipped, do we treat it as if it were absent and continue searching upward?
4. **Workspace root definition**
   * The workspace root is described as the directory containing a boundary like `.git` or `${APPNAME}_DIR`. If no boundary is found, is `workspace.path` `null` and we skip workspace checks entirely?
   * If multiple boundary markers exist (e.g., `.git` and `${APPNAME}_DIR` both present), which one takes precedence in determining the workspace root?
5. **Pattern matching implementation**
   * Globs are required for config and directory patterns, but third‑party dependencies are prohibited. Do we implement a minimal glob engine ourselves or is use of the `glob` or `minimatch` packages acceptable despite the "built‑ins" constraint?
   * Case sensitivity is optional. On Windows (case insensitive) how do we implement a case‑sensitive search when requested? This may require enumerating directory entries and comparing manually.
6. **Windows registry support**
   * Which module is recommended? The spec mentions `node-winreg` but marks it optional. Should the library gracefully fail if the module is not installed but `checkRegistry` is true?
   * Should registry reads be synchronous or asynchronous? (`node-winreg` uses callbacks.)
7. **Symlink handling**
   * For `symlinkBehavior: 'follow'`, we call `fs.realpath`. If a symlink crosses filesystem boundaries, does that affect boundary detection (e.g., mountpoints)?
   * For `as-is`, do we return the symlink path even if it points outside the workspace? The spec says to respect boundaries for symlinked paths, but details are sparse.
8. **Error handling**
   * The spec says to skip inaccessible paths without errors. Should an optional log or debug callback be provided for visibility?
   * What should happen if environment variables contain malformed values (e.g., empty string)? Simply ignore them?
9. **Long path support on Windows**
   * The spec mentions using the `\\?\` prefix for paths longer than 260 characters. Should the library automatically add this prefix or expect the caller to handle it?
10. **Return type nuance**
    * `findAppConfig` returns either a string, a `DiscoveryResult`, or `null`. Should the function always return `DiscoveryResult` when `select` includes more than `['configs']` or when `select` requests multiple scopes? Clarifying this would simplify usage.
11. **Cache / performance strategy**
    * The spec hints at caching `fs.stat` results. Is in‑memory caching across calls expected, or only within a single call? Should there be an option to disable caching?
12. **Test expectations**
    * Are there example fixtures or test cases planned to verify cross‑platform behavior, especially around boundary detection and pattern matching? This would guide the implementation.

## Potential Concerns

* **Complexity vs. minimal dependencies**: Implementing glob matching, case sensitivity, and symlink handling without third‑party libraries may become complex and error‑prone. Clarification on allowed dependencies would help.
* **Cross‑platform quirks**: Behavior like mountpoint detection or long path prefixes differs greatly between Linux and Windows. Thorough testing on both platforms may be challenging.
* **Return structure size**: The recursive pentad could grow large for deep directory trees (many parent directories). We may need limits or lazy evaluation to avoid large objects.
* **Edge cases**: Handling of unusual filesystems (e.g., network shares, WSL, path names with unusual characters) may require special logic beyond what is specified.

## Recommendations & Insights

1. **Clearly define environment variable handling**: Document how relative paths are resolved and how nonexistent files are treated.
2. **Consider always returning `DiscoveryResult`**: This would make the API consistent and easier to type, though it adds slight overhead when only one path is needed.
3. **Use the proven `glob` package** for pattern matching. This avoids reinventing the wheel while keeping the dependency footprint small.
4. **Provide utility functions** for common tasks such as expanding `~` to the home directory and normalizing paths. These can also aid cross-platform consistency.
5. **Expose debug logging hooks** to aid troubleshooting when discovery does not find expected files.
6. **Document expected behavior** for unusual scenarios (nested workspaces, symlink chains, invalid env vars) with examples.
7. **Testing**: Plan for extensive unit tests with mocked filesystems (e.g., using `memfs`) to simulate various platform behaviors without requiring actual Windows/Unix environments in CI.

## Conclusion

The spec presents a comprehensive roadmap for a cross‑platform configuration discovery library. Its goals are ambitious, especially given the restriction to Node.js built‑ins. Clarifying some of the points above—particularly around pattern matching, boundary precedence, and return types—would help ensure a smooth implementation. Overall, the design seems solid and should provide a flexible foundation for applications that need to locate configuration files in a standard‑compliant manner.


## Additional Recommendations

1. **Adopt the `glob` package** as the default pattern-matching implementation (see above).
2. **Centralize path utilities** such as `expandTilde`, `resolveRelative`, and `normalizeCase` to keep the code DRY and maintainable.
3. **Provide explicit examples** in the documentation for edge cases like nested workspaces and symlink loops. Concrete examples will reduce ambiguity.
4. **Define a minimal caching strategy** for `fs.stat` results to balance performance and simplicity. Document how to disable caching when deterministic behavior is required.
5. **Outline a clear testing matrix** covering Unix, macOS, and Windows behavior. This will help ensure cross-platform robustness early in development.
