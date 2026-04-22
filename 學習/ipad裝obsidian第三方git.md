# ipad 裝 obsidian 並同步 github 資料

-  **ipad 裝 obsidian**
	- 開始需要一個空的 vault (儲存庫)
	* 目前 ipad 我只設定拉下來，並不會修改。主要是想在外面時，可以看資料或分享資料。

- **GitHub 要取得 access token**
	1. 到 user 的 setting 頁面
	2. 最下方有一個 "Developer Settings"
	3. 看到 Personal access tokens\Fine-grained tokens 後，點入.
	4. 右上方有一個 "Generate new token"
	5. 填入資訊
		1. token name: <mark style="background:#fff88f">隨便</mark>
		2. expiration: <mark style="background:#fff88f">no expire</mark>
		3. permissions: add permissions
		   我有需 <mark style="background:#fff88f">contens, commit status, pull request</mark>.
		4. 產生的 token (PAT) 為 "github_pat_11AGESL6Y0uHeLNWP3XH6X_sLAvn7MMzOkPAnYK3hDKeptfyU1kUaMrw3wgqkLvpZIQDVA2X5Gzzemt5i"
		5. 只會出現一次，要保管好。
		
- **obsidian 設定第三方外掛 "Git"**
	1. 裝好後，需要啟用 Git
	2. cmd+P，打開快速鍵命令列，輸入 clone..應該就會出現 "Git: Clone on existing remote repo"
	3. 接著輸入 remote url
	   https://github.com/moyachang/markdown.git
	4. 大概類似以下這樣，後面就依照指示，填入 username, PAT等等的資訊。
	5. 完成後，它就會把 Github 上的資料拉下來。	 ![[Pasted image 20260422115219.png]]


