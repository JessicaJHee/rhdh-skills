# rhdh-patch-backstage workflow (Mermaid)

Companion diagram for [`SKILL.md`](../SKILL.md). Render in GitHub, GitLab, VS Code (Mermaid preview), or docs tooling.

## Overview

One flow: **backstage/backstage** at the **`backstage.json`** release tag produces **`dist/`**; **RHDH** pins that onto the **lockfile version** with **`yarn patch`**. Upstream SHAs are cherry-picked onto that tag when needed.

```mermaid
flowchart TB
  subgraph repos["Repos"]
    direction LR
    U(["backstage/backstage<br/><i>checkout v from RHDH backstage.json; optional cherry-pick; yarn build → dist/</i>"])
    R(["redhat-developer/rhdh<br/>release-V<br/><i>read backstage.json; yarn why/patch/patch-commit; verify resolutions</i>"])
  end

  subgraph work["Workflow"]
    direction TB
    P[Pre-flight: clean trees, map git remotes] --> S1[Sync RHDH → release-V; read backstage.json version]
    S1 --> S2[Fetch tags; checkout vVERSION on disposable branch]
    S2 --> S3{Bring fix onto tag}
    S3 -->|Upstream SHAs| CP[cherry-pick or skip if already in tag]
    S3 -->|manual edits| ME[direct changes in code]
    CP --> B[yarn build per package/plugin to generate dist/]
    ME --> B
    B --> RHDH[yarn why → yarn patch @npm:VERSION → copy dist → patch-commit]
    RHDH --> I[yarn install]
    I --> V[Verify yarn.lock + yarn why to confirm patch application]
  end
```

Step-by-step detail, troubleshooting, and **dynamic-plugins** (second Yarn project) are in [`SKILL.md`](../SKILL.md).
