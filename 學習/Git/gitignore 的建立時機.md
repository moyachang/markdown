## .gitignore 的設定順序

正確順序是：**先建立 `.gitignore`，再執行 `git add .`**

```bash
# 1. 初始化 git
git init

# 2. 建立並設定 .gitignore  ← 在 add 之前！
touch .gitignore
# （編輯 .gitignore，加入你不想追蹤的檔案）

# 3. 加入所有檔案（這時 .gitignore 已生效）
git add .

# 4. 第一次 commit
git commit -m "Initial commit"
```

---

## 為什麼順序很重要？

`git add .` 會把「當下還沒被忽略的檔案」全部加入 staging area。

如果你**先 `git add .` 再設定 `.gitignore`**，那些你不想要的檔案已經被 git 追蹤了，`.gitignore` 對**已追蹤的檔案無效**。

---

## 如果順序搞錯了怎麼補救？

```bash
# 把已追蹤的檔案從 git 移除（但保留本地檔案）
git rm -r --cached .

# 重新 add（這次 .gitignore 就會生效）
git add .

git commit -m "Apply .gitignore"
```

---

## 小提示

`.gitignore` 本身通常是**要** commit 上去的，這樣團隊其他人 clone 下來也會套用同樣的忽略規則。