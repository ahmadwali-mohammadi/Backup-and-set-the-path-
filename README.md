# Backup-and-set-the-path-


Great — glad it works. Here's a repeatable guide to add the same **"pick a folder and create a subfolder"** flow to any new Tauri + SvelteKit project.

### 🎯 What you're porting

Three things:
1. **Capabilities** — grant fs/dialog/path permissions with broad scope
2. **A DB/utility layer** — `getDb`, `moveDbTo`, etc.
3. **A `PathBackup.svelte`-style component** — folder picker + mkdir + save

### 📋 Step-by-step

---

#### 1️⃣ Create the new project

```bash
npx sv create my-new-app
cd my-new-app
npm install
npm run tauri init
```

Choose TypeScript, Skeleton, static adapter.

---

#### 2️⃣ Install the plugins you need

From project root:

```bash
npm run tauri add sql
npm run tauri add dialog
npm run tauri add fs
npm run tauri add opener
```

Each command adds both the Rust crate and the JS package, and registers the plugin in `src-tauri/src/lib.rs`.

---

#### 3️⃣ Confirm plugins in `lib.rs`

`src-tauri/src/lib.rs` should look like:

```rust
mod db;

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .plugin(
            tauri_plugin_sql::Builder::default()
                .add_migrations(db::DB_CONNECTION, db::migrations())
                .build(),
        )
        .plugin(tauri_plugin_dialog::init())
        .plugin(tauri_plugin_fs::init())
        .plugin(tauri_plugin_opener::init())
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

---

#### 4️⃣ Copy the capabilities file

Create **`src-tauri/capabilities/default.json`**:

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "Capability for the main window",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "core:path:default",
    "opener:default",
    "sql:default",
    "sql:allow-execute",
    "sql:allow-select",
    "sql:allow-load",
    "sql:allow-close",
    "dialog:default",
    "dialog:allow-open",

    { "identifier": "fs:allow-copy-file",  "allow": [{ "path": "**" }] },
    { "identifier": "fs:allow-exists",     "allow": [{ "path": "**" }] },
    { "identifier": "fs:allow-mkdir",      "allow": [{ "path": "**" }] },
    { "identifier": "fs:allow-remove",     "allow": [{ "path": "**" }] },
    { "identifier": "fs:allow-read-file",  "allow": [{ "path": "**" }] },
    { "identifier": "fs:allow-write-file", "allow": [{ "path": "**" }] }
  ]
}
```

---

#### 5️⃣ Add the folder-picker + mkdir utility

Create **`src/lib/folder.ts`** (framework-agnostic):

```ts
import { open } from "@tauri-apps/plugin-dialog";
import { exists, mkdir } from "@tauri-apps/plugin-fs";
import { downloadDir } from "@tauri-apps/api/path";

/** Ask the user to pick a folder, then create `subfolder` inside it. */
export async function pickFolderWithSub(subfolder: string): Promise<string | null> {
  const selected = await open({ directory: true, multiple: false });
  if (typeof selected !== "string") return null;

  const base = selected.replace(/[\\/]+$/, "");
  const alreadyNamed = new RegExp(`[\\\\/]${subfolder}$`, "i").test(base);
  const target = alreadyNamed ? base : `${base}\\${subfolder}`;

  if (!(await exists(target))) {
    await mkdir(target, { recursive: true });
  }

  return target;
}

/** Default: Downloads\<subfolder>. */
export async function defaultFolder(subfolder: string): Promise<string> {
  try {
    const downloads = await downloadDir();
    return `${downloads}\\${subfolder}`;
  } catch {
    return `C:\\Users\\Public\\${subfolder}`;
  }
}

/** Replace the drive letter of a Windows path: C:\x → E:\x */
export function switchDrive(path: string, drive: string): string {
  return path.replace(/^[A-Za-z]:/, drive);
}
```

That's the whole "browse + create subfolder + default + drive switch" logic in one file. Reusable in any project.

---

#### 6️⃣ Use it in a Svelte component

Create **`src/lib/components/PathPicker.svelte`** — the reusable UI:

```svelte
<script lang="ts">
  import Button from "$lib/components/ui/Button.svelte";
  import Icon from "$lib/components/ui/Icon.svelte";
  import { pickFolderWithSub, defaultFolder, switchDrive } from "$lib/folder";

  let {
    subfolder = "myapp",
    onConfirm
  }: { subfolder?: string; onConfirm: (folder: string) => Promise<void> | void } = $props();

  let folder = $state("");
  let error = $state("");
  let saving = $state(false);
  let loading = $state(true);

  const DRIVES = ["C:", "D:", "E:"];

  $effect(() => {
    (async () => {
      folder = await defaultFolder(subfolder);
      loading = false;
    })();
  });

  async function browse() {
    error = "";
    try {
      const picked = await pickFolderWithSub(subfolder);
      if (picked) folder = picked;
    } catch (e) {
      error = e instanceof Error ? e.message : String(e);
    }
  }

  async function confirm() {
    error = "";
    if (!folder.trim()) { error = "Choose a folder."; return; }
    saving = true;
    try {
      await onConfirm(folder.trim());
    } catch (e) {
      error = e instanceof Error ? e.message : String(e);
    } finally {
      saving = false;
    }
  }
</script>

<div class="row">
  <input class="input" bind:value={folder} placeholder="C:\Users\...\Downloads" disabled={loading} />
  <Button variant="secondary" type="button" onclick={browse} disabled={loading}>Browse</Button>
</div>

<div class="drives">
  {#each DRIVES as d (d)}
    <button type="button" class="drive" onclick={() => (folder = switchDrive(folder, d))}>
      <Icon name="server" size={16} />
      {d}
    </button>
  {/each}
</div>

{#if error}<div class="alert">{error}</div>{/if}

<Button type="button" onclick={confirm} disabled={loading || saving}>
  {saving ? "Saving…" : "Confirm"}
</Button>

<style>
  .row { display: flex; gap: 0.5rem; }
  .row .input { flex: 1; }
  .input { height: 36px; border-radius: 8px; border: 1px solid var(--slate-200); padding: 0 0.75rem; font-family: inherit; }
  .drives { display: flex; gap: 0.5rem; margin-top: 0.5rem; }
  .drive { display: inline-flex; align-items: center; gap: 0.35rem; padding: 0.4rem 0.75rem; border: 1px solid var(--slate-200); border-radius: 8px; background: #fff; cursor: pointer; font-family: inherit; }
  .alert { margin-top: 0.5rem; padding: 0.6rem 0.75rem; border-radius: 8px; background: #fef2f2; color: #dc2626; font-size: 0.8rem; }
</style>
```

Now any page can drop it in:

```svelte
<PathPicker subfolder="myapp" onConfirm={async (folder) => { await saveToDb(folder); }} />
```

---

#### 7️⃣ Wire it to your SQL plugin (if using SQL)

Create **`src/lib/db/database.ts`**:

```ts
import Database from "@tauri-apps/plugin-sql";
import { appDataDir, join } from "@tauri-apps/api/path";
import { copyFile, mkdir, exists } from "@tauri-apps/plugin-fs";

export const DB_CONNECTION = "sqlite:myapp.db";

let db: Database | null = null;
let initPromise: Promise<Database> | null = null;

export function getDb(): Promise<Database> {
  if (initPromise) return initPromise;
  initPromise = (async () => {
    db = await Database.load(DB_CONNECTION);
    return db;
  })();
  return initPromise;
}

export async function closeDb() {
  if (db) { await db.close(); db = null; initPromise = null; }
}

export async function moveDbTo(targetAbsPath: string) {
  await closeDb();
  const appData = await appDataDir();
  const source = await join(appData, "myapp.db");

  const lastSlash = Math.max(targetAbsPath.lastIndexOf("\\"), targetAbsPath.lastIndexOf("/"));
  const folder = lastSlash > 0 ? targetAbsPath.substring(0, lastSlash) : "";
  if (folder && !(await exists(folder))) {
    await mkdir(folder, { recursive: true });
  }

  await copyFile(source, targetAbsPath);
}
```

---

#### 8️⃣ Test

```bash
npm run tauri dev
```

Open the page with `<PathPicker />`, click **Browse**, pick any folder — the subfolder is created and the input updates. Click **Confirm** to hand the path back to your handler.

### 🎯 The 3 files you actually copy between projects

| File | Purpose |
|---|---|
| `src-tauri/capabilities/default.json` | fs/dialog/sql permissions |
| `src/lib/folder.ts` | `pickFolderWithSub`, `defaultFolder`, `switchDrive` |
| `src/lib/components/PathPicker.svelte` | UI (or roll your own) |

### ⚠️ Common gotchas when porting

| Issue | Cause | Fix |
|---|---|---|
| `forbidden path` | fs scope not set | Add `{ "path": "**" }` to each fs permission |
| `createDir not found` | Tauri 2 renamed it | Use `mkdir` |
| `downloadDir not found` | Wrong import | `@tauri-apps/api/path` (not a plugin) |
| `plugin not found: fs` | Plugin not registered | Check `lib.rs` has `.plugin(tauri_plugin_fs::init())` |
| Changes not applied | Capability is read at startup | Restart `npm run tauri dev` |

### 🎯 Optional — package it as a local npm module

If you reuse this across 3+ projects, extract `folder.ts` and `PathPicker.svelte` into a small npm package or a shared git repo, then:

```bash
npm install ../shared-tauri-ui
```

For 2–3 projects, copying the 3 files is faster.

### ✅ TL;DR

1. `npm run tauri add sql dialog fs opener`
2. Copy `capabilities/default.json` (broad fs scope)
3. Copy `src/lib/folder.ts`
4. Copy `PathPicker.svelte` and use it anywhere
5. Restart the dev server

That's the whole recipe. Want me to bundle `folder.ts` + `PathPicker.svelte` into a single `setup.ts` file that also handles the DB move, so it's one file to copy per project?
