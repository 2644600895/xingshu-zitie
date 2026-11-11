# 行书写法 · 字帖

一个以手机为主的纯静态网页：以「单个汉字」为唯一 key，收录该字的行书、草书、繁体写法图片；搜索文字即可查看，点图放大，可保存图片。增删改会**直接提交回 GitHub 仓库**，所有设备打开同一个页面看到的都是同一份数据。

## 它是怎么工作的

没有后端、没有 App、没有数据库。手机浏览器打开的就是 GitHub Pages 上的同一个网页：

- 读：页面加载时用 GitHub API 读仓库里的 `data.json`（有 token 时实时读仓库最新版，没 token 时读 Pages 上的同源文件，只读也能看）。
- 写：点保存时，先把新增图片以 base64 通过 `PUT /repos/{owner}/{repo}/contents/images/xxx.jpg` 提交，再把全量索引写回 `data.json`；写前会先取一次 `sha` 避免并发冲突。
- 同步：commit 落到 `main` 分支后触发 Actions 重新部署 Pages，其他设备一两分钟后就能看到。
- token：源码里是占位符 `__GH_TOKEN__`，Actions 部署时用仓库 Secret `GH_PAT` 替换。**真 token 只存在于部署出去的页面里，不会写回 git 历史**。

参考了你之前那个减肥打卡网站的思路（浏览器直连 GitHub API + Actions 注入最小权限 token + 单一数据源），差别在于：打卡站每次全量覆盖一个纯文本 `checkin.json`；这里的索引仍是全量覆盖（文本很小），但图片改成**只增量提交新增文件**，避免每次把所有图片重传一遍。

## 目录结构

```
index.html                      网页本体（内联 CSS / JS，零依赖）
data.json                       索引：文字 -> 图片路径
images/                         图片文件
.github/workflows/deploy.yml    部署并注入 token 的工作流
README.md                       本说明
```

## 发布步骤

1. 新建 GitHub 仓库，把整个目录提交并 push 到 `main`。`.github/` 目录也要提交。
2. 生成一个 **fine-grained Personal Access Token**：只在目标仓库下授权 **Contents: Read and write**，别给别的权限，也别给别的仓库。
3. 仓库 Settings → Secrets and variables → Actions → New repository secret，名字填 `GH_PAT`，值填上面那个 PAT。
4. 仓库 Settings → Pages → Build and deployment → Source 选 **GitHub Actions**（不要选 Deploy from a branch，因为 token 需要由工作流注入）。
5. 等首次工作流跑完，访问 `https://<用户名>.github.io/<仓库名>/`。

owner 和 repo 会从页面 URL 自动推断，不用填。本地预览时可在页面底部「设置」里临时填 owner / repo / PAT 试用增删改，这三项只存在浏览器 localStorage，不进仓库。

## data.json 格式

`chars` 是数组，每个元素一条记录，`char` 必填且唯一；`x` / `c` / `f` 分别是行书 / 草书 / 繁体图片数组，可以为空数组，元素里的 `src` 是相对 `index.html` 的路径：

```json
{
  "chars": [
    {
      "char": "永",
      "x": [{ "src": "images/img_1730000000000_1.jpg" }],
      "c": [],
      "f": [{ "src": "images/img_1730000000000_2.png" }]
    }
  ]
}
```

图片由网页自动命名提交（`img_<时间戳>_<序号>.<扩展名>`，纯 ASCII）。也可以手动往 `images/` 放图再改 `data.json`，但记得一起 commit + push。

## 增删改查

点右上角「＋ 添加」填字并选图片，保存时校验文字非空；三类图片都可留空、也可同类放多张；卡片上点「编辑」可改图、删单张或删整条；搜索框按字即时过滤。删除只移除索引里的引用，**不物理删除已上传的图片文件**（留着无害，也避免误删）。

每次保存会产生 1 到 2 次 commit（图片 + 索引），并触发一次 Pages 重新部署，属正常现象。提交失败时改动会自动回滚，并提示检查 owner / repo / PAT 权限与分支保护。

底部「重新载入」从仓库重拉最新数据；「导出备份」下载一份含图片数据的 JSON 用于本地存档，体积较大，**不要提交到仓库**。

## 关于 token 暴露（重要）

按你的判断：这个站基本自己用、内容也不敏感，所以接受下面这个固有取舍。

部署后的页面源码里包含 `GH_PAT` 的明文，任何访问该页面的人都能在「查看源代码」里看到它，并借它往这个仓库写东西。缓解措施：

- PAT 用 fine-grained，只授权这一个仓库的 Contents 读写，不给其他权限、不给其他仓库。
- 建议用一个专门的 bot / 小号签发，不要用自己的主账号全权限 token。
- 定期轮换 PAT，轮换后更新 Secret 并重新跑一次工作流。
- 不要给 `main` 开分支保护（否则网页的 PUT 会失败）。

如果你以后不想接受这一条，改动量也不大：加一层 Cloudflare Worker 持有 token，前端只调 `/api/save`，页面里就不再出现 token。

## 手机端要点

- 输入框字号 16px，避免 iOS 自动缩放；按钮和缩略图按触屏尺寸做。
- 点缩略图弹出大图，点「下载 / 保存图片」保存；部分手机浏览器对 `download` 支持有限，可长按大图选「保存图片」。
- 图片上传前建议压到单张几百 KB：走 API 是 base64，体积会膨胀约三分之一，仓库也会随图片增长。
- 不要直接双击 `index.html` 打开（`file://` 下读不到 `data.json`），走 Pages 或本地静态服务器。

## 本地预览

```bash
python -m http.server 8080
```

然后访问 `http://localhost:8080/`。此时没有注入 token，页面是只读的；想试增删改就点底部「设置」填上仓库信息和 PAT。
