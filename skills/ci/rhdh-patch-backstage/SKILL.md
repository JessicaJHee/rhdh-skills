---
name: rhdh-patch-backstage
description: >-
  Backports Backstage fixes into an RHDH release-* line without bumping published
  @backstage/* versions by checking out the Backstage release tag from RHDH's
  backstage.json in a local backstage/backstage clone, building package dist/,
  and generating Yarn patches under .yarn/patches with resolutions and lockfile
  updates in redhat-developer/rhdh (root and/or dynamic-plugins/). Use for
  "patch-backstage", "yarn patch this Backstage commit", "backport a CVE fix
  into release-1.9 with yarn patches", or building dist from the backstage.json
  tag onto the lockfile version. Not for cherry-picking an rhdh-plugins pull
  request onto a plugin release branch.
compatibility: "git, yarn (Berry), Node on PATH; gh when opening PRs; clean checkouts of redhat-developer/rhdh and backstage/backstage."
---

# RHDH Patch Backstage

Ship a fix on an RHDH **`release-*`** line **without** bumping published Backstage
versions by adding **Yarn patches** (`.yarn/patches/`, `package.json`
**`resolutions`**, lockfiles).

Read `workflows/overview.md` for the Mermaid diagram of the same flow.

## Purpose

- **`COMMITS`:** SHAs on **[backstage/backstage](https://github.com/backstage/backstage)** that are on **`master`** (merged fixes). Fetch **`master`** so those objects exist for **`git show`** / cherry-pick.
- **Build source:** a local **backstage/backstage** clone checked out at the release tag from RHDH **`backstage.json`** (see below). **Do not** use **redhat-developer/backstage** `patch/release-*`.
- **Cherry-pick** those SHAs onto that tag **only** if the **pre-check** shows the tag’s source still differs; otherwise **build + patch RHDH** only.

### What `backstage.json` pins

RHDH records only a **version string**, not a git commit. Example ([PR #4354](https://github.com/redhat-developer/rhdh/pull/4354) and current trees):

```json
{ "version": "1.54.9" }
```

That is the [Backstage release](https://github.com/backstage/backstage/releases) whose git tag is **`v<version>`** (e.g. **`v1.54.9`**). Resolve the tip commit with **`git rev-parse v<version>^{commit}`** after fetching tags. There is no separate commit field in `backstage.json`.

## Repos

| Repo | Role |
|------|------|
| **redhat-developer/rhdh** | Patches live here. Sync **`release-<RHDH_VERSION>`**; read **`backstage.json`**; run **`yarn patch`** (root and/or **`dynamic-plugins/`**). |
| **backstage/backstage** | Build tree: local clone. Check out **`v$(jq -r .version <RHDH_ROOT>/backstage.json)`**, optional cherry-pick of **`COMMITS`**, **`yarn build`** per package **`cd`**, copy **`dist/`** into RHDH patch temps. |

## Parameters

| Name | Required | Notes |
|------|----------|--------|
| **`RHDH_VERSION`** | Yes | e.g. `1.9` → **`release-1.9`**. Do not infer from the current branch. |
| **`RHDH_ROOT`** | No | Absolute path to the RHDH repo root. Inferred from context if omitted. |
| **`BACKSTAGE_ROOT`** | No | Absolute path to a local **backstage/backstage** clone. Inferred from context if omitted. |
| **`COMMITS`** | Typical | Upstream SHAs (oldest first for cherry-pick). If no SHAs, manual **`dist`** / patch only. |
| **`PACKAGES`** | If unclear | `@backstage/...` names. Derive from **`COMMITS`** (below) when paths map cleanly. |

Derived after Step 1:

- **`BACKSTAGE_VERSION`** = **`jq -r .version <RHDH_ROOT>/backstage.json`**
- **`BACKSTAGE_TAG`** = **`v<BACKSTAGE_VERSION>`**

**Path map:** `@backstage/plugin-<id>` → **`plugins/<id>/`**; other **`@backstage/<id>`** → **`packages/<id>/`**.

### Deriving **`PACKAGES`** from **`COMMITS`**

In the Backstage clone, after **`git fetch <BACKSTAGE_REMOTE> master`** (and tags) so **`COMMITS`** exist locally: **`git show --name-only --pretty=format: <SHA>`** (union for multiple SHAs). Map **`plugins/*`** and **`packages/*`** roots; read each **`package.json`** **`name`**; dedupe. Ignore-only changes (root lockfile, **`.changeset/`**, **`docs/`**, version-only **`package.json`**) → ask which packages to patch.

**Example:** `66e08b08f94a31cbf28b416c89b61549bc3b64a2` → **`@backstage/cli-common`**, **`@backstage/backend-plugin-api`**, **`@backstage/plugin-techdocs-node`**.

## Git remotes and hooks

**Map remotes by URL** in **each** clone (`git remote -v`); never assume **`upstream`** means a given org.

- **RHDH core** (`release-*`): URL **redhat-developer/rhdh** → **`RHDH_CORE_REMOTE`**.
- **Upstream Backstage** (build tree + **`COMMITS`**): URL **backstage/backstage** → **`BACKSTAGE_REMOTE`**.

**Silencing hooks:** For branch sync only (fetch/checkout/pull, and **`git cherry-pick`** when you are not relying on hook side effects), prefix with **`HUSKY=0`**. Omit **`HUSKY=0`** on **`git commit`** if you want **lint-staged** locally.

```bash
cd <RHDH_ROOT> && HUSKY=0 git fetch <RHDH_CORE_REMOTE> release-<V> && HUSKY=0 git checkout release-<V> && HUSKY=0 git pull <RHDH_CORE_REMOTE> release-<V>
```

## Agent execution

- **Batch** related shell commands with **`&&`** and **`cd <RHDH_ROOT>`** / **`cd <BACKSTAGE_ROOT>`**; cwd may not persist between tool calls.
- **Stop and ask** when **`RHDH_VERSION`**, **`COMMITS`**, clone paths, or workspace ownership is missing or ambiguous—not for a second confirmation when the user already asked for **yarn patches** for given SHAs (see **Pre-check → patch-only**).
- **Do not** invent remotes or wander with speculative **`find`**; **do** run steps this doc names (**`git remote -v`**, **`yarn why`**, **`jq`**, etc.).

## Dist baseline

- **`yarn patch`** overlays **`dist/`** on the **version RHDH already resolves** (lockfile), so **`PACKAGE_VERSION`** must come from **`yarn why`**, not from “what Backstage released.”
- **Compile** on **backstage/backstage** at **`BACKSTAGE_TAG`** (plus any cherry-picks). That tag is the release RHDH declares in **`backstage.json`**.
- **Do not** use **redhat-developer/backstage** **`patch/release-*`** as the build tree.

## Workflow (overview)

1. **Pre-flight:** Clean trees and remotes; set **`RHDH_CORE_REMOTE`**, **`BACKSTAGE_REMOTE`**.
2. **RHDH:** **`HUSKY=0`** fetch/checkout/pull **`release-<RHDH_VERSION>`**; read **`BACKSTAGE_VERSION`** / **`BACKSTAGE_TAG`** from **`backstage.json`**.
3. **Backstage:** Fetch tags + **`master`**; check out **`BACKSTAGE_TAG`** on a disposable work branch; **pre-check**; cherry-pick **or** patch-only; **`yarn build`** per **`PACKAGES`** (**`cd` + `yarn build`**, not root **`yarn workspace … build`**).
4. **RHDH:** Remove stale **`.patch`** + **`resolutions`** for targets; **`yarn why`** → versions; **`yarn patch`** / replace **`dist`** / **`patch-commit`**; **clean up `resolutions`**; **`yarn install`** (root and **`dynamic-plugins/`** as needed).
5. **Verify:** **`@patch:`** in each relevant **`yarn.lock`**; **`yarn why`** shows **`via patch:`**; commit artifacts.

Commits, pushes, and pull requests are external writes. Follow **`/mutation-gate`** with each repository and branch as the target (`rhdh:release-1.9`). Preview the artifact set before asking to commit or open a PR. Do not push the disposable Backstage work branch unless the user asks.

---

## Step 1: Sync RHDH

**Pre-flight (both repos):** **`git status`** clean in **RHDH** and the **Backstage** clone (stash WIP or **`git merge --abort`** / **`git rebase --abort`** as needed). Do not run the workflow mid-conflict.

1. **`git remote -v`** → **`RHDH_CORE_REMOTE`** = remote for **redhat-developer/rhdh**.
2. **`HUSKY=0 git fetch … release-<RHDH_VERSION>`** && **`HUSKY=0 git checkout …`** && **`HUSKY=0 git pull …`**. Fail if the branch is missing.
3. Read **`BACKSTAGE_VERSION`** / **`BACKSTAGE_TAG`** from **`<RHDH_ROOT>/backstage.json`**. Fail if the file or **`version`** field is missing.

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

- **Proceed** without extra confirmation if the user already asked for **yarn patches** for **`COMMITS`** on this **`release-*`**: empty pre-check → go to **2.5 Build** and RHDH Steps 3–5; note in the **Final summary** that cherry-pick was skipped.
- **Ask** if intent is vague (“sync Backstage” only).

### 2.4 Cherry-pick (when pre-check was non-empty)

**`git cherry-pick <SHA>`** (oldest first) onto **`rhdh-patch/<BACKSTAGE_TAG>`**.

**Conflicts:** Prefer the **cherry-picked commit’s** content for conflicted **`src/`** (during cherry-pick, **`git checkout --theirs -- <path>`** refers to that commit). If fixing conflicts would drop the functional fix, or conflicts are only **changelog/version** noise you should not merge, **`git cherry-pick --abort`**, note it in the **Final summary**, and coordinate with the user.

### 2.5 Build **`PACKAGES`**

For each package: **`cd`** **`plugins/<id>/`** or **`packages/<id>/`** → **`yarn build`**. If **`dist-types`** or build fails: Backstage repo root **`yarn install`** / **`yarn tsc`**, then retry per-package **`yarn build`**.

## Step 3: Prepare RHDH

**`dynamic-plugins/`** is a **separate** Yarn project: its **`yarn.lock`** and **`resolutions`** are independent of the root.

1. Remove prior **`.patch`** files and matching **`resolutions`** entries for the packages you are refreshing (search **both** root and **`dynamic-plugins/package.json`**).
2. **`yarn why @backstage/<pkg>`** from **RHDH root**; if empty, run from **`dynamic-plugins/`**. Record **PACKAGE_VERSION** for Step 4.

## Step 4: Generate patches

Run from **RHDH root** or **`dynamic-plugins/`** depending on where the dependency resolves.

**Temp folder:** Each **`yarn patch …`** prints a **new** path—use it immediately for **`rm`**, **`cp`**, **`yarn patch-commit -s`** (same shell or paste path). Do not reuse an old temp.

**Per package:**

1. **`yarn patch <package>@npm:<PACKAGE_VERSION>`**
2. **`rm -rf <PATCH_TEMP>/dist`** && **`cp -r <BACKSTAGE_ROOT>/<plugins|packages>/.../dist <PATCH_TEMP>/dist`**
3. **`yarn patch-commit -s <PATCH_TEMP>`** in that workspace.

If a package resolves in **both** trees at the **same** version, patch **both** places (or mirror the **`patch:`** resolution into **`dynamic-plugins/package.json`** and run **`yarn install`** there too).

### After **`yarn patch-commit`** (cleanup — required)

1. **Replace** any bare **`"@backstage/foo": "1.2.3"`** in **`resolutions`** with the **single** **`patch:`** locator Yarn printed—**do not** leave bare semver beside new patch keys (patch may not apply; Step 5 will show plain **`@npm:`**).
2. **Delete** spurious range keys **`patch-commit`** added (e.g. **`@backstage/foo@^1.6.0`** → patch built from **`@npm:1.5.0`**), which mis-resolve other workspaces (**`dynamic-plugins/`** may need **`@backstage/backend-plugin-api@1.6.0`** untouched by root).
3. **Install:** **`yarn install`** at root and/or **`dynamic-plugins/`**.

## Step 5: Verify (required)

Incomplete until every patched package shows a **patch locator** in **`yarn.lock`** and **`yarn why`**.

1. **`yarn install`** in each touched project.
2. **`grep '@backstage/<pkg>@patch' yarn.lock`** (run inside **`dynamic-plugins/`** for packages resolved only there).
3. **`yarn why @backstage/<pkg>`** in the same directory: must include **`via patch:`** / **`@…@patch:`**, not only **`via npm:<version>`**.

**Optional:** Spot-check **`node_modules/@backstage/<pkg>/dist/`**.

**Commit** only when the user asks, via **`/mutation-gate`**: **`.yarn/patches/*.patch`**, **`package.json`** **`resolutions`**, **`yarn.lock`** (root and/or **`dynamic-plugins/`**); PR to RHDH.

**PR CI:** a change under **`.yarn/patches/`** makes RHDH PR CI build and test every package (not **`--affected`**) and run the **Backstage bump checks** job. See [scripts/backstage-bump-check/README.md](https://github.com/redhat-developer/rhdh/blob/main/scripts/backstage-bump-check/README.md). It exercises the patched code only where existing tests reach it; E2E remains the integration gate.

## Completion

A run is complete when:

1. Links: `https://github.com/backstage/backstage/commit/<SHA>` for each **`COMMITS`** entry (or “manual dist / no SHAs”).
2. **`BACKSTAGE_TAG`** / tip commit from **`backstage.json`**; **`PACKAGES`**, patch locations (root vs **`dynamic-plugins/`**), notable **`resolutions`** keys.
3. Cherry-pick skipped vs applied; conflicts/aborts if any.
4. RHDH PR intent (or “not opened”).
5. **`RHDH_VERSION`**.
6. Every patched package shows **`@patch:`** / **`via patch:`** in the relevant **`yarn.lock`** and **`yarn why`**.

## Safety

- Do not invent a **`backstage.json`** version or a tag that does not exist on **backstage/backstage**.
- No blind conflict resolution on cherry-picks; no push of the disposable Backstage work branch unless the user asks.

## Common issues

| Symptom | What to do |
|---------|------------|
| Wrong build tip | Re-read **`backstage.json`**, fetch tags, **`checkout -B rhdh-patch/<tag> <tag>`** |
| Tag missing locally | **`git fetch <BACKSTAGE_REMOTE> --tags`** |
| **`bad object`** on cherry-pick / show | **`git fetch <BACKSTAGE_REMOTE> master`** |
| Checkout/pull fails after Yarn / Husky | **`HUSKY=0`** on those git commands |
| **`yarn why` / lockfile: no `@patch:`** | Remove bare semver **`resolutions`** for that pkg; use one **`patch:`** locator; drop wrong range keys; reinstall |
| Build fails (missing types) | Backstage root **`yarn install`** / **`yarn tsc`**, then **`cd`** package **`yarn build`** |
| Patch wrong version | Use **`yarn why`** version, not **`backstage.json`** meta-version |
