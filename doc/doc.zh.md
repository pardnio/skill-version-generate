# skill-version-generate - 技術文件

> 返回 [README](./README.zh.md)

## 前置需求

- Claude Code CLI（安裝於本機）
- `git` ≥ 2.23（使用 `git log` trailer 語法）
- 儲存庫已初始化 git（`.git/` 存在）
- `git config user.name` 與 `user.email` 必須設定（發版者驗證硬性條件）
- `gh` CLI（選用）— 啟用 GitHub handle 自動填入；未安裝或未登入時降級

## 安裝

### Clone 至 Claude Code 技能目錄

```bash
git clone https://github.com/pardnchiu/skill-version-generate ~/.claude/skills/version-generate
```

安裝後，Claude Code 會自動識別 `version-generate` 為可用技能。

### 驗證安裝

```bash
ls ~/.claude/skills/version-generate/SKILL.md
ls ~/.claude/skills/version-generate/scripts/
```

兩個路徑皆需存在。

## 設定

### 發版者身份

本技能讀取 `git config` 作為發版者身份，無額外設定檔。

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

若缺失任一欄位，Step 0 會中止並提示修復指令。

### GitHub Handle（可選）

若 `gh` CLI 已登入，frontmatter 的 `released_by.github` 會自動填入：

```bash
gh auth login
```

未登入時該欄位會省略，不中止流程。

## 使用方式

### 基礎

在專案目錄中呼叫：

```
/version-generate
```

執行流程：

1. 驗證 `git config user.name` / `user.email`
2. 取得最新標籤（`git tag -l 'v*.*.*' --sort=-v:refname | head -1`）
3. 解析 `$LATEST_TAG..HEAD` 的 commits（Conventional Commits 優先）
4. 依標籤優先權計算 `NEW_VERSION`
5. 產出 `.doc/NEW_VERSION.md`（含 frontmatter、追溯欄位）
6. Prepend 一行至 `.doc/CHANGELOG.md`

### 進階

#### 無標籤專案（首次發版）

若未曾打過標籤，`LATEST_TAG` 預設為 `v0.0.0`，`NEW_VERSION` 固定為 `v0.1.0`。

#### BREAKING 變更

commit subject 含 `!` 或 body 含 `BREAKING CHANGE:` trailer 即觸發 MAJOR 升版，並**強制要求 Migration 區段**。若 Migration 內容缺失，生成中止不留殘缺檔。

```
feat!: change AuthClient.Login signature
```

或

```
feat: change AuthClient.Login signature

BREAKING CHANGE: Login now requires context.Context and Credentials struct.
```

#### 僅 DOC / CHORE 變更

若 commits 僅含 `STYLE` / `DOC` / `TEST` / `CHORE` 類型，`type: none` 寫入 frontmatter，**不更新主索引**，保持目前版本號。

## 命令列參考

### 核心流程（`scripts/*.md` 實作細節）

| 步驟 | 檔案 | 職責 |
|---|---|---|
| 0 | `scripts/00-validate-releaser.md` | 驗證 `git config user.name` / `user.email`，選擇性抓取 `gh api user` |
| 1 | `scripts/01-detect-version.md` | 取得 `LATEST_TAG`、正規化 `REMOTE_URL` |
| 2 | `scripts/02-collect-changes.md` | Conventional Commits 解析 + diff fallback |
| 3 | `scripts/03-classify-and-bump.md` | 分類標籤與 SemVer 版號計算 |
| 4 | `scripts/04-output-template.md` | 產出 `.doc/NEW_VERSION.md` 完整模板 |
| 5 | `scripts/05-update-index.md` | Prepend 新版本至 `.doc/CHANGELOG.md` |
| ∞ | `scripts/06-rules-and-edge-cases.md` | 分類規則與邊界案例 |

### Conventional Commits 型別映射

| Conventional | 內部標籤 | 版本影響 |
|---|---|---|
| `feat` | FEAT | MINOR |
| `fix` | FIX | PATCH |
| `perf` | PERF | PATCH |
| `refactor` | REFACTOR | PATCH |
| `security` | SECURITY | PATCH |
| `revert` | REMOVE | PATCH |
| `docs` | DOC | — |
| `test` | TEST | — |
| `style` | STYLE | — |
| `chore` / `build` / `ci` | CHORE | — |
| 任一 type + `!` | 追加 BREAKING | MAJOR |

### SemVer 升版規則

| 條件 | `NEW_VERSION` |
|---|---|
| `LATEST_TAG = v0.0.0`（無標籤） | `v0.1.0` |
| 偵測到任一 `BREAKING` | `v(X+1).0.0` |
| 偵測到任一 `FEAT` | `vX.(Y+1).0` |
| 偵測到任一 `PATCH_TAGS` | `vX.Y.(Z+1)` |
| 僅 `NO_BUMP` 標籤 | `LATEST_TAG`（`type: none`） |

### Frontmatter 欄位

| 欄位 | 來源 | 必要 |
|---|---|---|
| `version` | `NEW_VERSION` | ✅ |
| `previous` | `LATEST_TAG` | ✅ |
| `date` | 今日日期（`YYYY-MM-DD`） | ✅ |
| `type` | `major` / `minor` / `patch` / `none` | ✅ |
| `breaking` | `true` / `false` | ✅ |
| `released_by.name` | `git config user.name` | ✅ |
| `released_by.email` | `git config user.email` | ✅ |
| `released_by.github` | `gh api user --jq .login` | 缺失省略 |
| `contributors` | `git log $LATEST_TAG..HEAD --format='%an <%ae>' \| sort -u` | ✅ |
| `co_authors` | `Co-authored-by` trailer | 空則省略 |
| `compare` | `$REMOTE_URL/compare/$LATEST_TAG...$NEW_VERSION` | 非 GitHub 省略 |

### 條目格式

每條變更尾綴 `(#PR, @author) [short_hash]`，缺失欄位省略對應括號：

| 完整度 | 範例 |
|---|---|
| 完整 | `Add OAuth2 login flow (#142, @alice) [a3f9c2d]` |
| 無 PR | `Add OAuth2 login flow (@alice) [a3f9c2d]` |
| 無 handle | `Add OAuth2 login flow (#142) [a3f9c2d]` |
| 最小 | `Add OAuth2 login flow [a3f9c2d]` |

### 輸出位置

| 路徑 | 內容 |
|---|---|
| `.doc/vX.Y.Z.md` | 單版本完整 changelog |
| `.doc/CHANGELOG.md` | 主索引（最新在最上） |

### 中止條件

| 條件 | 行為 |
|---|---|
| `git config user.name` 或 `user.email` 缺失 | Step 0 中止 |
| `BREAKING` 變更無 Migration 內容 | Step 4 中止 |
| `LATEST_TAG..HEAD` 無 commits | 輸出「無變更」訊息並結束 |
