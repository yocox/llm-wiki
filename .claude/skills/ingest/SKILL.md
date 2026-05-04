---
name: ingest
description: 將 raw/ 目錄下的原始資料編譯到 wiki/ 中。處理完成後，將源檔案自動移動到 raw/09-archive/ 歸檔。支援 `/ingest` (掃描 raw/ 下所有未歸檔檔案) 或 `/ingest <path>` (處理指定檔案)。當使用者提到「攝取」、「匯入」、「收入」資料，或要求將檔案加入知識庫時，也應該觸發此技能。絕對忽略 raw/09-archive/ 目錄。
user-invocable: true
---

# ingest 技能

## 核心工作流程：收件匣與歸檔

你正在維護一個 **LLM Wiki**（Obsidian 知識庫）。`raw/` 目錄是「待處理收件匣」，`wiki/` 是「編譯輸出層」。

**目錄結構約定：**
- `raw/01-articles/` — 網頁剪藏的 Markdown 文章
- `raw/02-papers/` — 論文和 PDF 文獻
- `raw/03-transcripts/` — 影片逐字稿
- `raw/09-archive/` — **已處理檔案的歸檔目錄，禁止讀取**
- `wiki/sources/` — 資料摘要
- `wiki/entities/` — 實體（人物、公司、工具、產品）
- `wiki/concepts/` — 概念（框架、方法論、理論）

## 觸發邏輯

1. **使用者執行 `/ingest`**：掃描 `raw/` 所有子目錄（排除 `09-archive/`），找出待處理檔案。
2. **使用者執行 `/ingest <path>`**：僅處理指定檔案。
3. **隱式觸發**：使用者說「把這個資料攝入知識庫」、「匯入這篇文章」時，自動執行 ingest。

## 編譯流水線

對每個待處理源檔案，嚴格按以下步驟執行：

### 步驟 1：讀取源檔案

- **如果是 `.md` 檔案**：使用讀取工具完整讀取內容。
- **如果是 `.pdf` 檔案**：使用讀取工具嘗試提取文字。如果無法提取或內容為空，改為記錄檔案元資訊（檔案名稱、頁數）在 sources 頁面中。

### 步驟 2：提煉核心並翻譯

從源檔案中提取：
- **核心主旨**：這份資料講什麼（1-2 句話）
- **實體**：人物、公司、工具、產品等具體名詞
- **概念**：框架、方法論、理論等抽象名詞

如果是非中文內容，則翻譯成繁體中文。

### 步驟 3：建立來源摘要

在 `wiki/sources/` 建立 Markdown 檔案：

```markdown
---
title: "摘要-文件slug"
type: source
tags: [來源, 原始檔案]
sources: [raw/01-articles/xxx.md]
last_updated: YYYY-MM-DD
---

## 核心摘要
[3-5 句話的核心總結]

## 關聯連結
- [[EntityName]] — 關聯實體
- [[ConceptName]] — 關聯概念
```

檔案名稱使用 kebab-case：`摘要-{文件slug}.md`

### 步驟 4：知識網路化（實體 / 概念頁面）

對於步驟 2 提取的每個實體和概念：

**目標目錄：**
- 實體 → `wiki/entities/`
- 概念 → `wiki/concepts/`

**處理邏輯：**
1. 頁面不存在 → 按照 CLAUDE.md 的 Frontmatter 規範建立新頁面
2. 頁面已存在 → 讀取現有內容，**增量合併**新資訊
3. **發現衝突** → **立即暫停**，向使用者回報衝突內容，詢問處理方式後再繼續

**頁面範本：**

```markdown
---
title: "頁面名稱"
type: entity | concept
tags: [標籤]
sources:
  - "[[摘要-source-slug]]"
last_updated: YYYY-MM-DD
---

## 定義
[對該實體 / 概念的定義]

## 關鍵資訊
[從源檔案中提取的詳細資訊]

## 關聯連結
- [[摘要-source-slug]] — 來源
- [[RelatedEntity]] — 相關實體
```

**Frontmatter sources 格式規範**：

✅ **正確格式**（YAML 列表 + Obsidian wikilinks）：
```yaml
sources:
  - "[[Effective harnesses for long-running agents]]"
  - "[[2210.17323v2.pdf]]"
  - "[[Harness design for long-running application development]]"
```

**关键规则**：
- ✅ **Markdown 檔案**（`.md`）：不含副檔名 → `[[頁面標題]]`
- ✅ **PDF 檔案**（`.pdf`）：**必須包含**副檔名 → `[[檔名.pdf]]`
- ✅ **雙重方括弧**：`[[...]]` Obsidian wikilink 格式
- ✅ **字符串用雙引號**：`"[[...]]"` YAML 規範
- ❌ **不用路徑**：不需要 `raw/09-archive/` 前綴
- ❌ **不混用格式**：不要用陣列 `[...]` 代替列表

### 步驟 5：更新全域登錄表

**更新 `wiki/index.md`：**
按照 CLAUDE.md 規定的格式，將新增頁面加入對應分類下：
- Sources: `[[摘要-source-slug]] — 該資料的核心主旨`
- Entities: `[[EntityName]] — 該實體的身份定義`
- Concepts: `[[ConceptName]] — 該概念的核心定義`

**更新 `wiki/log.md`：**
追加操作日誌（Append-only）：
```markdown
## [YYYY-MM-DD] ingest | 操作簡述
- **變更**: 新增 [[PageName]]; 更新 [[index.md]]
- **衝突**: 無 (或: 衝突 [[ConflictingPage]]，已暫停等待決策)
```

### 步驟 6：歸檔源檔案（使用 obsidian-cli 維護連結）

**重要**：使用 `obsidian-cli` 進行檔案移動，以自動更新所有指向該檔案的 frontmatter links。

**流程**：
1. 確認以下全部完成：
   - sources 頁面已建立
   - 實體 / 概念頁面已建立或更新
   - index.md 已更新
   - log.md 已更新

2. **使用 obsidian-cli 移動檔案**：
   ```powershell
   obsidian-cli move-note "raw/01-articles/源檔案.md" "raw/09-archive/源檔案.md"
   ```

**Frontmatter 中的 links 最佳實踐**：

使用 **Obsidian wikilinks 格式 + YAML 列表**：
```yaml
sources:
  - "[[Markdown 檔案名稱]]"         # .md 不含副檔名
  - "[[PDF_檔案名稱.pdf]]"          # .pdf 必須含副檔名
  - "[[另一個檔案]]"
```

**詳細規則**：
| 檔案類型 | 範例 | 說明 |
|---------|------|------|
| `.md` Markdown | `[[Effective harnesses for long-running agents]]` | 不含副檔名 |
| `.pdf` PDF | `[[2210.17323v2.pdf]]` | **必須包含** `.pdf` |
| `.txt` 文本 | `[[filename.txt]]` | 其他格式包含副檔名 |

**絕對禁止**：
- ❌ 混合格式：`sources: [[[page]], "path/file.pdf"]`
- ❌ 包含路徑：`[[raw/09-archive/file]]`（只需檔案名）
- ❌ Markdown 加副檔名：`[[file.md]]`（應為 `[[file]]`）
- ❌ 修改源檔案內部的文字
- ❌ 使用簡單的 `mv` 命令移動檔案（會破壞 Obsidian 連結）

## 衝突處理流程

當發現新舊知識衝突時：

1. **暫停**：停止當前 ingest 流程
2. **回報**：向使用者說明衝突內容（哪個頁面、衝突點是什麼）
3. **詢問**：請使用者選擇處理方式：
   - A) 保留新舊兩者，標注為「知識衝突」
   - B) 以新知識覆蓋舊知識
   - C) 放棄本次 ingest
4. **繼續**：根據使用者選擇繼續或終止

## 注意事項

- 絕對不讀取 `raw/09-archive/` 下的任何檔案
- 所有 wiki 頁面必須包含 `## 關聯連結` 區域，不能產生孤立頁面
- 使用繁體中文編寫所有內容
- 實體命名使用 TitleCase，概念和來源使用 kebab-case
- Frontmatter 當中的 links 要記得使用 `[[頁面名稱]]` 的形式建立 obsidian internal link，有多個 links 的話，就以 multiline list 格式
