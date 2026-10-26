# rhdh-patch-backstage workflow (Mermaid)

Companion diagram for [`SKILL.md`](../SKILL.md). Render in GitHub, GitLab, VS Code (Mermaid preview), or docs tooling.

## Overview

**backstage/backstage** at the **`backstage.json`** tag is built twice; only differing dist files become the Yarn patch in **rhdh**. The same **`src/`** fix must also land as a tracking PR on **redhat-developer/backstage** **`patch/release-V`**. That tracking PR is a **merge gate** for the RHDH PR.

```mermaid
flowchart TB
  subgraph repos["Repos"]
    direction LR
    U(["backstage/backstage<br/><i>build tree: tag + cherry-pick; baseline vs patched dist</i>"])
    M(["redhat-developer/backstage<br/>patch/release-V<br/><i>tracking PR with same src/ fix — required before RHDH merge</i>"])
    R(["redhat-developer/rhdh<br/>release-V<br/><i>minimal yarn patch; PR links maintenance PR</i>"])
  end

  subgraph work["Workflow"]
    direction TB
    P[Pre-flight: clean trees, map git remotes] --> S1[Sync RHDH → release-V; read backstage.json]
    S1 --> S2[Fetch tags; checkout vVERSION]
    S2 --> B0[yarn build → DIST_BASELINE]
    B0 --> S3{Apply fix}
    S3 -->|Upstream SHAs| CP[cherry-pick onto tag]
    S3 -->|manual edits| ME[edit src/]
    CP --> B1[yarn build → DIST_PATCHED]
    ME --> B1
    B1 --> D[diff baseline vs patched; exclude *.map]
    D --> T[Sync patch/release-V; same src/; open tracking PR]
    T --> RHDH[yarn patch → copy only changed files → patch-commit]
    RHDH --> I[yarn install + verify]
    I --> G{Maintenance PR open or merged?}
    G -->|yes| MOK[RHDH PR may merge]
    G -->|no| BLOCK[Block RHDH merge]
  end
```

Step-by-step detail, troubleshooting, and **dynamic-plugins** (second Yarn project) are in [`SKILL.md`](../SKILL.md).
