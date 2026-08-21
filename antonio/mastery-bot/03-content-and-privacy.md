# Content & Privacy

## Canonical paths

Every path anywhere in the app is a **POSIX-style, content-root-relative string with no leading/trailing slash** — `"networking-mastery/03-tcp.md"`, or `""` for the root. This single convention (`src/content/paths.ts`) is what lets the same path travel safely through Telegram `callback_data`, GitHub API calls, and the embeddings index without ever touching a real OS filesystem path or a URL-encoded form.

`normalizeRelativePath()` is the choke point every untrusted path-like string passes through — from a model's proposed save location, from decoded `callback_data`, from a user-typed folder name. It rejects traversal (`..`), absolute paths, drive letters, null bytes, and back-slashes, and is re-applied *after* resolving `..` segments specifically so a check like `path.startsWith(editorFolder)` can't be defeated by something like `"antonio/../someone-else/notes.md"` (see [Save & Authoring](06-save-authoring.md) for where this matters most: never trusting a path an LLM generated).

## `ContentProvider`: one interface, two implementations

```ts
export interface ContentProvider {
  listDirectory(path: string): Promise<ContentEntry[]>;
  getDocument(path: string): Promise<Document>;
  search(query: string): Promise<SearchResult[]>;
}
```

- **`LocalFilesystemContentProvider`** — reads from a local folder (`CONTENT_ROOT`), used for local development and for the offline `build:index` script.
- **`GitHubContentProvider`** — reads from the `mastery` repo via GitHub's Contents API, used in production.

`createContentProvider(env)` (`src/content/createContentProvider.ts`) is the single composition point that picks between them based on `CONTENT_PROVIDER`; nothing else in the app ever imports a concrete provider or checks that env var itself.

## Reading from GitHub

`GitHubApiClient` (`src/content/github/GitHubApiClient.ts`) is a minimal typed wrapper over the parts of GitHub's REST API the app actually needs: the Contents API (`getContents`, `getFileContent` — following `download_url` for files too large to inline), the Git ref API (`getBranchHeadSha`), and (for search) the recursive Git Trees API (`getTree`) plus `getBlob` for reading matched content by SHA rather than re-resolving a path. Every response is validated with a Zod schema before use — GitHub's API is trusted to be *reachable*, never trusted to return exactly the shape expected.

## Writing to GitHub

Writing is a **separate class**, `GitHubContentWriter`, kept apart from `GitHubContentProvider` deliberately: only `/save` needs write access, only for configured editors, and every read-side provider (including `GitHubContentProvider` itself) stays read-only. It exposes three operations:

```ts
write(path, content, message): Promise<{ path, beforeCommitSha }>
delete(path, message): Promise<{ path, beforeCommitSha }>
revert(path, beforeCommitSha, message): Promise<void>
```

The `beforeCommitSha` returned by `write`/`delete` is the branch's HEAD commit **immediately before that specific change** — captured so a later `revert()` can restore *just that one path* to what it looked like at that moment, without needing any server-side record of what happened. `revert()` is a **corrective commit, not a history rewrite**: it fetches the content that existed at `beforeCommitSha` (via a ref-scoped Contents API read) and either restores it (if it existed) or deletes the path (if it didn't) — the same "encode just enough to recompute" philosophy as the rest of the app's stateless design, applied to undo.

One subtlety that only surfaced once `delete()` was added (for the reorganize feature — see [Save & Authoring](06-save-authoring.md)): `revert()` originally treated "the path currently doesn't exist" as "already reverted, nothing to do" — correct when undoing a *write*, but wrong when undoing a *delete*, where "doesn't currently exist" is exactly the state that needs undoing. The fix distinguishes "didn't exist before the commit being reverted" from "doesn't exist right now," and recreates the file (via a create, not an update — GitHub rejects a create that passes a blob `sha`) when those two facts disagree. See [Lessons & Bugs](08-lessons-and-bugs.md) for the full story.

## `PRIVATE_FOLDERS`: per-folder read privacy

A single mechanism enforces "this top-level folder is only visible to its owner" everywhere content can be read — browsing, search, and RAG retrieval for `/ask` alike:

```ts
// src/content/visibility.ts
export function isPathVisible(
  canonicalPath: string,
  userId: number | undefined,
  privateFolders: readonly PrivateFolderConfig[],
): boolean {
  const topLevel = canonicalPath.split("/")[0];
  const owner = privateFolders.find((f) => f.folder === topLevel)?.ownerId;
  return owner === undefined || owner === userId;
}
```

`PRIVATE_FOLDERS` is a comma-separated `<folder-name>:<telegram-user-id>` list, independent of `EDITORS` — a folder can be private without being anyone's `/save` target (legacy content), or writable without being private. Every place that could leak a private document's existence — directory listings, search results, RAG-retrieved chunks, direct document-open callbacks — filters through `isPathVisible()`, and a private path someone isn't allowed to see returns the *same* "not found" response a genuinely missing path would (never a distinct "forbidden," which would itself leak that the path exists).

## `EDITORS`: per-user write confinement

Separately, `EDITORS` (`<telegram-user-id>:<folder-name>` pairs) determines who can use `/save` at all, and confines each editor's writes to their own single top-level folder — `findEditorFolder(userId, editors)` is the one function every write-path handler (`/save`, revert, the reorganize confirm flow) calls first, and every path a model or a user proposes is checked to start with that folder before anything is written, moved, or deleted.

## Why this design

Both mechanisms exist because the content repo evolved from "one person's notes" into "four people's notes in one repo" mid-project. Rather than four separate repos (more infrastructure, harder to reason about) or a database-backed permissions system (contradicts the stateless design), a folder convention plus two small, independently-testable predicate functions — one gating reads, one gating writes — turned out to be enough: the four users' folders (`antonio/`, `andreea/`, `anatol/`, `ana/`) are symmetric, each private to its owner, each writable only by its owner, with zero special-casing needed anywhere else in the codebase.
