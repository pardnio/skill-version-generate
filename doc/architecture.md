# skill-version-generate - Architecture

> Back to [README](../README.md)

## Overview

```mermaid
graph TB
    User[User runs /version-generate] --> Skill[SKILL.md<br/>contract and index]
    Skill --> S0[Step 0<br/>Releaser validation]
    S0 --> S1[Step 1<br/>Version detection]
    S1 --> S2[Step 2<br/>Change collection]
    S2 --> S3[Step 3-4<br/>Classify and bump]
    S3 --> S4[Step 5<br/>Emit changelog]
    S4 --> S5[Step 6<br/>Update master index]
    S4 --> OutA[.doc/version-generate/vX.Y.Z.md]
    S5 --> OutB[.doc/version-generate/CHANGELOG.md]
    Rules[scripts/06<br/>Rules and edge cases] -.applies.-> S2
    Rules -.applies.-> S3
    Rules -.applies.-> S4
```

## Module: SKILL.md (contract layer)

Defines the workflow, hard preconditions, SemVer mapping, and execution order. Contains no implementation details; execution is delegated to `scripts/*.md`.

```mermaid
graph LR
    subgraph SKILL.md
        Contract[Core contract] --> Precondition[Hard preconditions]
        Contract --> Output[Output spec]
        Contract --> Mapping[SemVer mapping]
        Contract --> Order[Execution order]
    end
    Contract -.indexes.-> Scripts[scripts/*.md]
```

## Module: Step 0 — Releaser validation

```mermaid
graph TB
    subgraph Step0
        Name[git config user.name] --> Check{missing?}
        Email[git config user.email] --> Check
        Check -- yes --> Abort[abort exit 1]
        Check -- no --> GH[gh auth status]
        GH -- authenticated --> Handle[gh api user .login]
        GH -- unauthenticated --> Skip[omit github field]
        Handle --> Pass[pass]
        Skip --> Pass
    end
```

## Module: Step 2 — Change collection (parser)

```mermaid
graph TB
    subgraph Collector
        GitLog[git log TAG..HEAD] --> Parse{Subject matches<br/>Conventional regex?}
        Parse -- yes --> CC[Conventional parser]
        Parse -- no --> Diff[Diff semantic analysis<br/>fallback]
        CC --> Tag[Internal tag]
        Diff --> Tag
        Subject[Subject !] --> Breaking[BREAKING flag]
        Body[Body BREAKING CHANGE:] --> Breaking
        Footer[trailing #123] --> PR[PR number]
        Trailer[Co-authored-by] --> CoAuthor[co_authors list]
    end
```

## Module: Step 3-4 — Classification and version bump

```mermaid
graph TB
    subgraph Classifier
        Tags[Tag set] --> Priority{Priority}
        Priority -- has BREAKING --> Major[MAJOR +1.0.0]
        Priority -- has FEAT --> Minor[MINOR +0.1.0]
        Priority -- has PATCH_TAGS --> Patch[PATCH +0.0.1]
        Priority -- only NO_BUMP --> None[keep LATEST_TAG]
        Major --> NewVer[NEW_VERSION]
        Minor --> NewVer
        Patch --> NewVer
        None --> NewVer
    end
```

## Module: Step 5 — Output template

```mermaid
graph TB
    subgraph Output
        Vars[Collect variables<br/>NEW_VERSION LATEST_TAG<br/>RELEASER CONTRIBUTORS] --> FM[Frontmatter assembly]
        FM --> Summary[Summary section]
        Summary --> Brk{breaking?}
        Brk -- yes --> Mig[Migration section<br/>abort if missing]
        Brk -- no --> Skip[omit section]
        Mig --> Changes[Changes section]
        Skip --> Changes
        Changes --> Scope[Scope section]
        Scope --> Write[write .doc/version-generate/vX.Y.Z.md]
    end
```

## Module: Step 6 — Master index maintenance

```mermaid
graph LR
    subgraph IndexUpdater
        Exist{.doc/version-generate/CHANGELOG.md<br/>exists?} -- no --> Init[initialize<br/>title + preamble]
        Exist -- yes --> Skip[preserve]
        Init --> Prepend
        Skip --> Prepend[prepend new version line]
        Prepend --> Stats[summary counts<br/>exclude NO_BUMP]
    end
```

## Data Flow

```mermaid
sequenceDiagram
    participant User
    participant Skill as SKILL.md
    participant Git
    participant GH as gh CLI
    participant FS as Filesystem

    User->>Skill: /version-generate
    Skill->>Git: git config user.name/email
    Git-->>Skill: releaser identity
    alt identity missing
        Skill-->>User: ERROR + exit 1
    end
    Skill->>GH: gh auth status
    GH-->>Skill: authenticated / not
    Skill->>Git: git tag -l --sort=-v:refname
    Git-->>Skill: LATEST_TAG
    Skill->>Git: git log LATEST_TAG..HEAD
    Git-->>Skill: commits + trailers
    Skill->>Skill: parse + classify + bump
    alt breaking without migration
        Skill-->>User: ERROR + abort
    end
    Skill->>FS: write .doc/version-generate/NEW_VERSION.md
    Skill->>FS: prepend .doc/version-generate/CHANGELOG.md
    FS-->>User: release artifacts ready
```

## State Machine (release flow)

```mermaid
stateDiagram-v2
    [*] --> Validating
    Validating --> Aborted: config missing
    Validating --> Detecting: pass
    Detecting --> Collecting
    Collecting --> Classifying
    Classifying --> NoBumpOnly: only NO_BUMP
    Classifying --> Composing: bump required
    Composing --> Aborted: BREAKING without Migration
    Composing --> Writing
    Writing --> Indexing
    Indexing --> Done
    NoBumpOnly --> Done: type=none, skip index
    Done --> [*]
    Aborted --> [*]
```
