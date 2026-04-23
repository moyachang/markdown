在 Obsidian 中限制 Mermaid 圖表的大小，最有效的方法是透過 **CSS Snippets（CSS 程式碼片段）**，因為 Mermaid 預設會根據容器寬度自動縮放，有時會變得過大或模糊。

你可以根據需求選擇以下幾種調整方式：

---

### 1. 全域限制（推薦）

如果你希望所有的 Mermaid 圖表都不要超過特定寬度，並保持居中，這是最簡單的方法。

1. 開啟 Obsidian 的 **Settings** > **Appearance**。
    
2. 找到 **CSS snippets**，點擊右側的資料夾圖示開啟資料夾。
    
3. 建立一個新的 `.css` 檔案（例如 `mermaid-style.css`），貼入以下代碼：
    

CSS

```css
/* 限制 Mermaid 寬度並居中 */
.mermaid {
    max-width: 600px; /* 你可以依照喜好調整數值 */
    margin: 0 auto;
    display: flex;
    justify-content: center;
}

/* 確保 SVG 內容能自適應縮放 */
.mermaid svg {
    max-width: 100%;
    height: auto;
}
```

4. 回到 Obsidian 重新整理 Snippets 列表，並**開啟**該開關。
    

---

### 2. 使用 HTML 標籤手動包裹（彈性高）

如果你只想針對「特定」某張圖表做限制，不想動到全域設定，可以直接在 Markdown 裡用 `div` 標籤包起來：

HTML


```text
<div style="width: 400px; margin: 0 auto;">
```

中間包 mermaid.

HTML

```
</div>
```

---

### 3. 針對不同類型的圖表微調

有時候流程圖（Graph）和甘特圖（Gantt）需要的寬度不同，你可以利用 CSS 選擇器進行細分：

CSS

```css
/* 只限制流程圖的寬度 */
.mermaid[data-type="graph"] svg {
    max-width: 400px;
}

/* 讓序列圖可以寬一點 */
.mermaid[data-type="sequence"] svg {
    max-width: 800px;
}
```

---

### 4. 解決「字體太小」的問題

當你限制了 `max-width`，有時候 Mermaid 會為了塞進去而讓字體縮得很小。這時可以嘗試在 Mermaid 代碼內部調整：

程式碼片段

```
%%{init: {'themeVariables': { 'fontSize': '20px'}}}%%
graph TD
    A --> B
```

### 小撇步

- **即時檢查**：調整 CSS 時，建議開啟一個分欄顯示該圖表，儲存 CSS 檔案後 Obsidian 通常會立即反映變化。
    
- **匯出 PDF**：CSS Snippets 對於匯出 PDF 同樣有效，能避免圖表在紙張頁面上爆框。