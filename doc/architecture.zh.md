# skill-version-generate - 架構

> 返回 [README](./README.zh.md)

## 概覽

```mermaid
graph TB
    User[使用者觸發 /version-generate] --> Skill[SKILL.md<br/>契約與索引]
    Skill --> S0[Step 0<br/>發版者驗證]
    S0 --> S1[Step 1<br/>版本偵測]
    S1 --> S2[Step 2<br/>收集變更]
    S2 --> S3[Step 3-4<br/>分類與版號計算]
    S3 --> S4[Step 5<br/>輸出 changelog]
    S4 --> S5[Step 6<br/>更新主索引]
    S4 --> OutA[.doc/vX.Y.Z.md]
    S5 --> OutB[.doc/CHANGELOG.md]
    Rules[scripts/06<br/>分類規則與邊界案例] -.套用.-> S2
    Rules -.套用.-> S3
    Rules -.套用.-> S4
```

## Module: SKILL.md（契約層）

定義工作流程、硬性前置條件、SemVer 映射表與執行順序。不含實作細節，實作委派給 `scripts/*.md`。

```mermaid
graph LR
    subgraph SKILL.md
        Contract[核心契約] --> Precondition[硬性前置條件]
        Contract --> Output[輸出規格]
        Contract --> Mapping[SemVer 映射]
        Contract --> Order[執行順序]
    end
    Contract -.索引.-> Scripts[scripts/*.md]
```

## Module: Step 0 — 發版者驗證

```mermaid
graph TB
    subgraph Step0
        Name[git config user.name] --> Check{缺失?}
        Email[git config user.email] --> Check
        Check -- 是 --> Abort[中止 exit 1]
        Check -- 否 --> GH[gh auth status]
        GH -- 已登入 --> Handle[gh api user .login]
        GH -- 未登入 --> Skip[省略 github 欄位]
        Handle --> Pass[通過]
        Skip --> Pass
    end
```

## Module: Step 2 — 收集變更（解析器）

```mermaid
graph TB
    subgraph Collector
        GitLog[git log TAG..HEAD] --> Parse{Subject 符合<br/>Conventional 正則?}
        Parse -- 是 --> CC[Conventional 解析器]
        Parse -- 否 --> Diff[Diff 語意分析<br/>fallback]
        CC --> Tag[內部標籤]
        Diff --> Tag
        Subject[Subject !] --> Breaking[BREAKING 旗標]
        Body[Body BREAKING CHANGE:] --> Breaking
        Footer[尾綴 #123] --> PR[PR 編號]
        Trailer[Co-authored-by] --> CoAuthor[co_authors 清單]
    end
```

## Module: Step 3-4 — 分類與版號

```mermaid
graph TB
    subgraph Classifier
        Tags[標籤集合] --> Priority{優先權}
        Priority -- 含 BREAKING --> Major[MAJOR +1.0.0]
        Priority -- 含 FEAT --> Minor[MINOR +0.1.0]
        Priority -- 含 PATCH_TAGS --> Patch[PATCH +0.0.1]
        Priority -- 僅 NO_BUMP --> None[保持 LATEST_TAG]
        Major --> NewVer[NEW_VERSION]
        Minor --> NewVer
        Patch --> NewVer
        None --> NewVer
    end
```

## Module: Step 5 — 輸出模板

```mermaid
graph TB
    subgraph Output
        Vars[收集變數<br/>NEW_VERSION LATEST_TAG<br/>RELEASER CONTRIBUTORS] --> FM[Frontmatter 組裝]
        FM --> Summary[Summary 區段]
        Summary --> Brk{breaking?}
        Brk -- 是 --> Mig[Migration 區段<br/>缺失則中止]
        Brk -- 否 --> Skip[省略整段]
        Mig --> Changes[Changes 區段]
        Skip --> Changes
        Changes --> Scope[Scope 區段]
        Scope --> Write[寫入 .doc/vX.Y.Z.md]
    end
```

## Module: Step 6 — 主索引維護

```mermaid
graph LR
    subgraph IndexUpdater
        Exist{.doc/CHANGELOG.md<br/>存在?} -- 否 --> Init[初始化<br/>標題 + 說明]
        Exist -- 是 --> Skip[保留]
        Init --> Prepend
        Skip --> Prepend[Prepend 新版本行]
        Prepend --> Stats[摘要統計<br/>排除 NO_BUMP]
    end
```

## 資料流

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
    GH-->>Skill: login / unauthenticated
    Skill->>Git: git tag -l --sort=-v:refname
    Git-->>Skill: LATEST_TAG
    Skill->>Git: git log LATEST_TAG..HEAD
    Git-->>Skill: commits + trailers
    Skill->>Skill: 解析 + 分類 + 版號計算
    alt breaking 無 migration
        Skill-->>User: ERROR + abort
    end
    Skill->>FS: write .doc/NEW_VERSION.md
    Skill->>FS: prepend .doc/CHANGELOG.md
    FS-->>User: 發版文件就緒
```

## 狀態機（發版流程）

```mermaid
stateDiagram-v2
    [*] --> Validating
    Validating --> Aborted: config 缺失
    Validating --> Detecting: 通過
    Detecting --> Collecting
    Collecting --> Classifying
    Classifying --> NoBumpOnly: 僅 NO_BUMP
    Classifying --> Composing: 有升版
    Composing --> Aborted: BREAKING 無 Migration
    Composing --> Writing
    Writing --> Indexing
    Indexing --> Done
    NoBumpOnly --> Done: type=none 不更新索引
    Done --> [*]
    Aborted --> [*]
```
