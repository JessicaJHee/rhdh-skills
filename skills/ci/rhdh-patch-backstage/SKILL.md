---
name: rhdh-patch-backstage
description: >-
  Backports Backstage fixes into an RHDH release-* line without bumping published
  @backstage/* versions by checking out the Backstage release tag from RHDH's
  backstage.json in a local backstage/backstage clone, building a minimal Yarn
  patch, and opening a tracking PR on redhat-developer/backstage patch/release-*
  that must exist before the RHDH patch PR may merge. Use for "patch-backstage",
  "yarn patch this Backstage commit", "backport a CVE fix into release-1.9 with
  yarn patches", or a minimal dist patch from the backstage.json tag. Not for
  cherry-picking an rhdh-plugins pull request onto a plugin release branch.
compatibility: "git, yarn (Berry), Node on PATH; gh when opening PRs; clean checkouts of redhat-developer/rhdh, backstage/backstage, and redhat-developer/backstage (or a fork of it)."
---

# RHDH Patch Backstage

Ship a fix on an RHDH **`release-*`** line **without** bumping published Backstage
versions by adding **Yarn patches** (`.yarn/patches/`, `package.json`
**`resolutions`**, lockfiles), and record the same source change on the
**redhat-developer/backstage** maintenance line for that release.

Read `workflows/overview.md` for the Mermaid diagram of the same flow.

## Purpose

- **`COMMITS`:** SHAs on **[backstage/backstage](https://github.com/backstage/backstage)** that are on **`master`** (merged fixes). Fetch **`master`** so those objects exist for **`git show`** / cherry-pick.
- **Build source:** a local **backstage/backstage** clone checked out at the release tag from RHDH **`backstage.json`** (see below). Use this tree for baseline/patched **`yarn build`** and the minimal Yarn patch.
- **Tracking (required):** the same functional **`src/`** change must be contributed as a PR to **[redhat-developer/backstage](https://github.com/redhat-developer/backstage)** base **`patch/release-<RHDH_VERSION>`**, so each RHDH release has an auditable patch history. This is a **merge gate** for the RHDH PR (see below).
- **Cherry-pick** those SHAs onto the build tag **only** if the **pre-check** shows the tag’s source still differs; otherwise there is nothing to patch or track.

### What `backstage.json` pins

RHDH records only a **version string**, not a git commit. Example ([PR #4354](https://github.com/redhat-developer/rhdh/pull/4354) and current trees):

```json
{ "version": "1.54.9" }
```

That is the [Backstage release](https://github.com/backstage/backstage/releases) whose git tag is **`v<version>`** (e.g. **`v1.54.9`**). Resolve the tip commit with **`git rev-parse v<version>^{commit}`** after fetching tags. There is no separate commit field in `backstage.json`.

## Repos

| Repo | Role |
|------|------|
| **redhat-developer/rhdh** | Yarn patches live here. Sync **`release-<RHDH_VERSION>`**; read **`backstage.json`**; run **`yarn patch`** (root and/or **`dynamic-plugins/`**). |
| **backstage/backstage** | Build tree: local clone. Check out **`v$(jq -r .version <RHDH_ROOT>/backstage.json)`**, optional cherry-pick of **`COMMITS`**, baseline + patched **`yarn build`**, minimal file copy into RHDH patch temps. |
| **redhat-developer/backstage** | Maintenance / tracking fork: **`patch/release-<RHDH_VERSION>`**. Same **`src/`** fix as a PR (from your fork). **Required** before the RHDH patch PR may merge. Not the Yarn-patch build tree. |

## Merge gate (required)

The RHDH Yarn-patch PR **must not merge** (to **`main`** or **`release-*`**) until a tracking PR against **redhat-developer/backstage** base **`MAINTENANCE_BRANCH`** (`patch/release-<RHDH_VERSION>`) is **open** (or already merged) with the same functional source change.

- Put the maintenance PR URL in the RHDH PR body and in the **Final summary**.
- If the user asks to merge the RHDH PR and the maintenance PR is missing, **stop**—open or locate that PR first.
- Prefer opening the maintenance PR **before** (or immediately with) the RHDH PR; never treat the RHDH PR as merge-ready without it.

## Parameters

| Name | Required | Notes |
|------|----------|--------|
| **`RHDH_VERSION`** | Yes | e.g. `1.9` → **`release-1.9`**, **`patch/release-1.9`**. Do not infer from the current branch. |
| **`RHDH_ROOT`** | No | Absolute path to the RHDH repo root. Inferred from context if omitted. |
| **`BACKSTAGE_ROOT`** | No | Absolute path to a local **backstage/backstage** clone (build tree). Inferred from context if omitted. |
| **`MAINTENANCE_ROOT`** | No | Absolute path to a local **redhat-developer/backstage** clone (or your fork). Inferred from context if omitted. |
| **`COMMITS`** | Typical | Upstream SHAs (oldest first for cherry-pick). If no SHAs, manual **`src/`** edits. |
| **`PACKAGES`** | If unclear | `@backstage/...` names. Derive from **`COMMITS`** (below) when paths map cleanly. |

Derived after Step 1:

- **`BACKSTAGE_VERSION`** = **`jq -r .version <RHDH_ROOT>/backstage.json`**
- **`BACKSTAGE_TAG`** = **`v<BACKSTAGE_VERSION>`**
- **`MAINTENANCE_BRANCH`** = **`patch/release-<RHDH_VERSION>`**

**Path map:** `@backstage/plugin-<id>` → **`plugins/<id>/`**; other **`@backstage/<id>`** → **`packages/<id>/`**.

### Deriving **`PACKAGES`** from **`COMMITS`**

In the Backstage clone, after **`git fetch <BACKSTAGE_REMOTE> master`** (and tags) so **`COMMITS`** exist locally: **`git show --name-only --pretty=format: <SHA>`** (union for multiple SHAs). Map **`plugins/*`** and **`packages/*`** roots; read each **`package.json`** **`name`**; dedupe. Ignore-only changes (root lockfile, **`.changeset/`**, **`docs/`**, version-only **`package.json`**) → ask which packages to patch.

**Example:** `66e08b08f94a31cbf28b416c89b61549bc3b64a2` → **`@backstage/cli-common`**, **`@backstage/backend-plugin-api`**, **`@backstage/plugin-techdocs-node`**.

## Git remotes and hooks

**Map remotes by URL** in **each** clone (`git remote -v`); never assume **`upstream`** means a given org.

- **RHDH core** (`release-*`): URL **redhat-developer/rhdh** → **`RHDH_CORE_REMOTE`**.
- **Upstream Backstage** (build tree + **`COMMITS`**): URL **backstage/backstage** → **`BACKSTAGE_REMOTE`**.
- **Maintenance Backstage** (`patch/release-*`, tracking PR): URL **redhat-developer/backstage** → usually your fork as **`origin`** / **`FORK_REMOTE`** for push; redhat-developer as the PR base remote.

**Silencing hooks:** For branch sync only (fetch/checkout/pull, and **`git cherry-pick`** when you are not relying on hook side effects), prefix with **`HUSKY=0`**. Omit **`HUSKY=0`** on **`git commit`** if you want **lint-staged** locally.

```bash
cd <RHDH_ROOT> && HUSKY=0 git fetch <RHDH_CORE_REMOTE> release-<V> && HUSKY=0 git checkout release-<V> && HUSKY=0 git pull <RHDH_CORE_REMOTE> release-<V>
```

## Agent execution

- **Batch** related shell commands with **`&&`** and **`cd <RHDH_ROOT>`** / **`cd <BACKSTAGE_ROOT>`**; cwd may not persist between tool calls.
- **Stop and ask** when **`RHDH_VERSION`**, **`COMMITS`**, clone paths, or workspace ownership is missing or ambiguous—not for a second confirmation when the user already asked for **yarn patches** for given SHAs (see **Pre-check → patch-only**).
- **Do not** invent remotes or wander with speculative **`find`**; **do** run steps this doc names (**`git remote -v`**, **`yarn why`**, **`jq`**, etc.).

## Dist baseline

- **`yarn patch`** overlays files on the **version RHDH already resolves** (lockfile), so **`PACKAGE_VERSION`** must come from **`yarn why`**, not from “what Backstage released.”
- **Compile** on **backstage/backstage** at **`BACKSTAGE_TAG`**, then again after the fix. The Yarn patch must contain **only** files that differ between those two local builds (the SHA / manual change)—never a wholesale **`dist/`** replace.
- **Do not** use **redhat-developer/backstage** **`patch/release-*`** as the Yarn-patch **build** tree; that clone is for the **tracking PR** only.

## Minimal patch rule (required)

Copying an entire rebuilt **`dist/`** into the Yarn patch temp pulls rebuild noise (sourcemaps, key reorder in **`.d.ts`**, toolchain path diffs). That produced oversized patches such as [rhdh@3e2c6fff](https://github.com/redhat-developer/rhdh/commit/3e2c6fff9ce2f36988bd44a89752ec63b01c0ec2) (~1.2k lines for a two-file SignInPage fix).

**Always:**

1. Build at **`BACKSTAGE_TAG`** → **`DIST_BASELINE`**.
2. Apply **`COMMITS`** / manual edits; rebuild → **`DIST_PATCHED`**.
3. Diff the two trees; copy **only** changed runtime artifacts into the Yarn patch temp (default: exclude **`*.map`**).
4. Abort and ask if the changed-file list is large or unrelated to the source paths from the SHA / manual edit.

## Workflow (overview)

1. **Pre-flight:** Clean trees and remotes; set **`RHDH_CORE_REMOTE`**, **`BACKSTAGE_REMOTE`**, maintenance remotes / **`FORK_REMOTE`**.
2. **RHDH:** **`HUSKY=0`** fetch/checkout/pull **`release-<RHDH_VERSION>`**; read **`BACKSTAGE_VERSION`** / **`BACKSTAGE_TAG`**; set **`MAINTENANCE_BRANCH`**.
3. **Build tree (`backstage/backstage`):** Fetch tags + **`master`**; check out **`BACKSTAGE_TAG`**; **baseline build**; **pre-check**; cherry-pick **or** manual edits; **patched build**; **diff dist trees**.
4. **Tracking (`redhat-developer/backstage`) — required:** Sync **`MAINTENANCE_BRANCH`**; apply the **same** functional **`src/`** change; push to **`FORK_REMOTE`**; open PR to **redhat-developer/backstage** base **`MAINTENANCE_BRANCH`**. Record the PR URL.
5. **RHDH Yarn patch:** Remove stale **`.patch`** + **`resolutions`**; **`yarn why`** → versions; **`yarn patch`**; copy **only** diffed files; **`patch-commit`**; install; verify.
6. **RHDH PR:** Open only with the maintenance PR linked; **do not merge** until that tracking PR is open or merged.

Commits, pushes, and pull requests are external writes. Follow **`/mutation-gate`** with each repository and branch as the target (`rhdh:release-1.9`, `backstage:patch/release-1.9`). Preview the artifact set before asking to commit or open a PR.

---

## Step 1: Sync RHDH

**Pre-flight:** **`git status`** clean in **RHDH**, the **backstage/backstage** build clone, and the **maintenance** clone (stash WIP or **`git merge --abort`** / **`git rebase --abort`** as needed). Do not run the workflow mid-conflict.

1. **`git remote -v`** → **`RHDH_CORE_REMOTE`** = remote for **redhat-developer/rhdh**.
2. **`HUSKY=0 git fetch … release-<RHDH_VERSION>`** && **`HUSKY=0 git checkout …`** && **`HUSKY=0 git pull …`**. Fail if the branch is missing.
3. Read **`BACKSTAGE_VERSION`** / **`BACKSTAGE_TAG`** from **`<RHDH_ROOT>/backstage.json`**. Fail if the file or **`version`** field is missing.
4. Set **`MAINTENANCE_BRANCH`** = **`patch/release-<RHDH_VERSION>`**.

## Step 2: Backstage clone (`backstage/backstage`)

**One** local clone of **backstage/backstage** (**`BACKSTAGE_ROOT`**).

### 2.1 Check out the `backstage.json` tag

```bash
cd <BACKSTAGE_ROOT> \
  && git fetch <BACKSTAGE_REMOTE> --tags \
  && git fetch <BACKSTAGE_REMOTE> master \
  && HUSKY=0 git checkout -B rhdh-patch/<BACKSTAGE_TAG> <BACKSTAGE_TAG>
```

Confirm **`git rev-parse HEAD`** matches **`git rev-parse <BACKSTAGE_TAG>^{commit}`** before cherry-picks. **`COMMITS`** should be reachable from **`master`** after the fetch.

### 2.2 Pre-check (skip cherry-pick when source already matches)

For each SHA in **`COMMITS`** (oldest first):

1. **`git show --name-only --pretty=format: <SHA>`**
2. **Drop** bookkeeping-only paths: root **`package.json`**, **`CHANGELOG.md`**, package **`package.json`** version-only edits, **`.changeset/`**, **`docs/`**, lockfiles, etc. **Keep** **`src/`**, tests, fixtures tied to the fix.
3. **`git diff <SHA> HEAD -- <kept paths>`** (union paths if multiple SHAs).

**Empty diff** → the release tag already has the functional fix; **do not cherry-pick** (avoids changelog/version noise).

### 2.3 Patch-only vs ask

- **Proceed** without extra confirmation if the user already asked for **yarn patches** for **`COMMITS`** on this **`release-*`**: empty pre-check and no manual edits → stop (nothing to patch); note that in the **Final summary**.
- If they still want a patch despite an empty pre-check, ask why (the tag already has the fix).
- **Ask** if intent is vague (“sync Backstage” only).

### 2.4 Baseline build (before the fix)

On **`rhdh-patch/<BACKSTAGE_TAG>`** at the clean tag tip (no cherry-picks yet), for each package in **`PACKAGES`**:

```bash
cd <BACKSTAGE_ROOT>/<plugins|packages>/<id> && yarn build \
  && rm -rf /tmp/rhdh-patch-baseline/<pkg-name> \
  && mkdir -p /tmp/rhdh-patch-baseline \
  && cp -a dist /tmp/rhdh-patch-baseline/<pkg-name>
```

If **`dist-types`** or build fails: Backstage repo root **`yarn install`** / **`yarn tsc`**, then retry. Record **`DIST_BASELINE=<pkg>`** = **`/tmp/rhdh-patch-baseline/<pkg-name>`**.

When the pre-check was empty (fix already in the tag) and there are no manual edits, stop—there is nothing to patch.

### 2.5 Cherry-pick (when pre-check was non-empty)

**`git cherry-pick <SHA>`** (oldest first) onto **`rhdh-patch/<BACKSTAGE_TAG>`**.

**Conflicts:** Prefer the **cherry-picked commit’s** content for conflicted **`src/`** (during cherry-pick, **`git checkout --theirs -- <path>`** refers to that commit). If fixing conflicts would drop the functional fix, or conflicts are only **changelog/version** noise you should not merge, **`git cherry-pick --abort`**, note it in the **Final summary**, and coordinate with the user.

For **manual edits** instead of SHAs: edit **`src/`** on this branch after the baseline build, then continue.

### 2.6 Patched build + dist diff

For each package: **`cd`** package → **`yarn build`** again. Then list files that differ from the baseline (**exclude sourcemaps by default**):

```bash
PKG_DIR=<BACKSTAGE_ROOT>/<plugins|packages>/<id>
BASE=/tmp/rhdh-patch-baseline/<pkg-name>
OUT=/tmp/rhdh-patch-changed-<pkg-name>.txt
: > "$OUT"
# Changed files present on both sides (skip *.map)
diff -rq "$BASE" "$PKG_DIR/dist" | grep ' differ$' | grep -v '\.map' | while IFS= read -r line; do
  left=${line#Files }
  left=${left%% and *}
  printf '%s\n' "${left#"$BASE"/}"
done >> "$OUT"
# Added under patched dist
diff -rq "$BASE" "$PKG_DIR/dist" | grep "^Only in $PKG_DIR/dist" | grep -v '\.map' | while IFS= read -r line; do
  rest=${line#Only in }
  dir=${rest%%: *}
  name=${rest#*: }
  printf '%s\n' "${dir#"$PKG_DIR/dist"/}/$name" | sed 's#^/##'
done >> "$OUT"
sort -u "$OUT" -o "$OUT"
cat "$OUT"
```

**Sanity check:** The relative paths should map to the **`src/`** files from **`COMMITS`** / the manual edit (e.g. `SignInPage.tsx` → `layout/SignInPage/SignInPage.esm.js`). If dozens of unrelated files appear, the baseline was wrong or the build is non-deterministic—**stop and ask**; do not copy the whole tree.

## Step 3: Tracking PR on `redhat-developer/backstage` (required)

Keep a per-release record of the source fix on **`MAINTENANCE_BRANCH`**. This step is **not** optional when Steps 2.5–2.6 produced a real change.

### 3.1 Sync `patch/release-*`

In **`MAINTENANCE_ROOT`** (map remotes; do not assume names):

```bash
cd <MAINTENANCE_ROOT> \
  && git fetch https://github.com/redhat-developer/backstage.git <MAINTENANCE_BRANCH> \
  && HUSKY=0 git checkout -B <MAINTENANCE_BRANCH> FETCH_HEAD
```

Fail if **`MAINTENANCE_BRANCH`** is missing on **redhat-developer/backstage**—do not invent it.

### 3.2 Apply the same functional `src/` change

Cherry-pick the same **`COMMITS`** (oldest first) or replay the same manual **`src/`** edits as on the build tree. Prefer the cherry-picked commit’s content for conflicted **`src/`**. Drop changelog/version-only noise when it is not needed for tracking. Do **not** rely on this tree for the Yarn **`dist/`** build.

### 3.3 Push and open the tracking PR

1. Push to **`FORK_REMOTE`** (your fork)—**no** direct push to **redhat-developer**.
2. Open a PR: **base** = **redhat-developer/backstage** **`MAINTENANCE_BRANCH`**, **head** = your fork branch. Use **`/mutation-gate`** before push/PR.
3. Record **`MAINTENANCE_PR_URL`**. Link upstream **`COMMITS`** in the PR body.

**Gate:** Without **`MAINTENANCE_PR_URL`** (open or already merged for this fix), do not mark the RHDH work merge-ready and do not merge the RHDH PR.

## Step 4: Prepare RHDH

**`dynamic-plugins/`** is a **separate** Yarn project: its **`yarn.lock`** and **`resolutions`** are independent of the root.

1. Remove prior **`.patch`** files and matching **`resolutions`** entries for the packages you are refreshing (search **both** root and **`dynamic-plugins/package.json`**).
2. **`yarn why @backstage/<pkg>`** from **RHDH root**; if empty, run from **`dynamic-plugins/`**. Record **PACKAGE_VERSION** for Step 4.

## Step 5: Generate patches (minimal file set)

Run from **RHDH root** or **`dynamic-plugins/`** depending on where the dependency resolves.

**Temp folder:** Each **`yarn patch …`** prints a **new** path—use it immediately. Do not reuse an old temp.

**Per package:**

1. **`yarn patch <package>@npm:<PACKAGE_VERSION>`** → **`PATCH_TEMP`**.
2. **Do not** **`rm -rf <PATCH_TEMP>/dist`** or copy the entire built **`dist/`**.
3. For each relative path in **`/tmp/rhdh-patch-changed-<pkg-name>.txt`**:

   ```bash
   cp "$PKG_DIR/dist/$rel" "$PATCH_TEMP/dist/$rel"
   ```

   Create parent dirs in **`PATCH_TEMP`** if a new file was added.
4. **`yarn patch-commit -s <PATCH_TEMP>`** in that workspace.
5. **Inspect** the new **`.yarn/patches/*.patch`**: it must list only the intended paths (typically a handful of **`.js`** / **`.d.ts`** under the changed feature). If it is dominated by **`*.map`** or unrelated modules, delete it and redo Step 2.4–2.6 and Step 5.

If a package resolves in **both** trees at the **same** version, apply the **same minimal file set** in both places (or mirror the **`patch:`** resolution into **`dynamic-plugins/package.json`** and run **`yarn install`** there too).

### After **`yarn patch-commit`** (cleanup — required)

1. **Replace** any bare **`"@backstage/foo": "1.2.3"`** in **`resolutions`** with the **single** **`patch:`** locator Yarn printed—**do not** leave bare semver beside new patch keys (patch may not apply; Step 5 will show plain **`@npm:`**).
2. **Delete** spurious range keys **`patch-commit`** added (e.g. **`@backstage/foo@^1.6.0`** → patch built from **`@npm:1.5.0`**), which mis-resolve other workspaces (**`dynamic-plugins/`** may need **`@backstage/backend-plugin-api@1.6.0`** untouched by root).
3. **Install:** **`yarn install`** at root and/or **`dynamic-plugins/`**.

## Step 6: Verify (required)

Incomplete until every patched package shows a **patch locator** in **`yarn.lock`** and **`yarn why`**, and **`MAINTENANCE_PR_URL`** is set.

1. **`yarn install`** in each touched project.
2. **`grep '@backstage/<pkg>@patch' yarn.lock`** (run inside **`dynamic-plugins/`** for packages resolved only there).
3. **`yarn why @backstage/<pkg>`** in the same directory: must include **`via patch:`** / **`@…@patch:`**, not only **`via npm:<version>`**.
4. Confirm **`MAINTENANCE_PR_URL`** is an open or merged PR to **redhat-developer/backstage** **`MAINTENANCE_BRANCH`**.

**Optional:** Spot-check **`node_modules/@backstage/<pkg>/dist/`**.

**Commit** only when the user asks, via **`/mutation-gate`**: **`.yarn/patches/*.patch`**, **`package.json`** **`resolutions`**, **`yarn.lock`** (root and/or **`dynamic-plugins/`**); PR to RHDH. The RHDH PR body **must** link **`MAINTENANCE_PR_URL`**. State explicitly that the RHDH PR is **not mergeable** until that tracking PR is open or merged.

**PR CI:** a change under **`.yarn/patches/`** makes RHDH PR CI build and test every package (not **`--affected`**) and run the **Backstage bump checks** job. See [scripts/backstage-bump-check/README.md](https://github.com/redhat-developer/rhdh/blob/main/scripts/backstage-bump-check/README.md). It exercises the patched code only where existing tests reach it; E2E remains the integration gate.

## Completion

A run is complete when:

1. Links: `https://github.com/backstage/backstage/commit/<SHA>` for each **`COMMITS`** entry (or “manual edits / no SHAs”).
2. **`BACKSTAGE_TAG`** / tip commit from **`backstage.json`**; **`PACKAGES`**, patch locations (root vs **`dynamic-plugins/`**), notable **`resolutions`** keys.
3. The list of copied dist relative paths (from the baseline↔patched diff), and confirmation the **`.patch`** file contains only those paths.
4. Cherry-pick skipped vs applied; conflicts/aborts if any.
5. **`MAINTENANCE_BRANCH`**, **`MAINTENANCE_PR_URL`** (required when a fix was applied).
6. RHDH PR URL (if opened) with the maintenance PR linked; merge status: blocked until tracking PR exists.
7. **`RHDH_VERSION`**.
8. Every patched package shows **`@patch:`** / **`via patch:`** in the relevant **`yarn.lock`** and **`yarn why`**.

## Safety

- Do not invent a **`backstage.json`** version, a tag that does not exist on **backstage/backstage**, or a missing **`patch/release-*`** on **redhat-developer/backstage**.
- No blind conflict resolution on cherry-picks; no direct push to **redhat-developer** remotes.
- Never commit a Yarn patch that replaces an entire **`dist/`** or is mostly **`*.map`** / unrelated modules.
- Never merge (or advise merging) the RHDH Yarn-patch PR without an open or merged maintenance tracking PR for the same fix.

## Common issues

| Symptom | What to do |
|---------|------------|
| Huge **`.patch`** (maps / whole dist) | Redo with baseline vs patched dist diff; copy only changed files; exclude **`*.map`** |
| Wrong build tip | Re-read **`backstage.json`**, fetch tags, **`checkout -B rhdh-patch/<tag> <tag>`** |
| Tag missing locally | **`git fetch <BACKSTAGE_REMOTE> --tags`** |
| **`bad object`** on cherry-pick / show | **`git fetch <BACKSTAGE_REMOTE> master`** |
| **`patch/release-*` missing** | Confirm the branch on **redhat-developer/backstage**; do not invent it |
| RHDH PR ready but no tracking PR | Open Step 3 PR first; link it; only then consider RHDH merge |
| Checkout/pull fails after Yarn / Husky | **`HUSKY=0`** on those git commands |
| **`yarn why` / lockfile: no `@patch:`** | Remove bare semver **`resolutions`** for that pkg; use one **`patch:`** locator; drop wrong range keys; reinstall |
| Build fails (missing types) | Backstage root **`yarn install`** / **`yarn tsc`**, then **`cd`** package **`yarn build`** |
| Patch wrong version | Use **`yarn why`** version, not **`backstage.json`** meta-version |
| Dist diff empty after cherry-pick | Confirm cherry-pick touched **`src/`** that this package builds; rebuild; check **`PACKAGES`** mapping |
