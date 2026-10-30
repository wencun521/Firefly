# Pages CMS 后台使用说明

站点内容通过 [Pages CMS](https://pagescms.org) 托管后台管理，所有修改以 commit 形式写回本仓库，再由 Vercel 自动构建部署。

## 首次接入

1. 打开 https://app.pagescms.org 并使用 GitHub 登录
2. 按提示安装 Pages CMS 的 GitHub App，授权范围选择 `wencun521/Firefly`
3. 在仓库列表中选择本仓库，后台会读取根目录的 `.pages.yml` 生成界面

## 可管理的内容

| 模块 | 对应文件 | 说明 |
| --- | --- | --- |
| 文章 | `src/content/posts/*.md` | 新建、编辑、删除文章与 frontmatter |
| 关于页面 | `src/content/spec/about.md` | 编辑正文 |
| 站点信息 | `src/config/site.json` | 标题、副标题、URL、描述、关键词、建站日期、主题色相 |
| 媒体库 | `public/images/posts/` | 上传封面与正文图片 |

其余配置（导航、侧栏、评论、社交链接等）仍在 `src/config/` 下的 TypeScript 文件中维护。

## 文章字段

标题、发布日期、描述、封面、标签、分类、草稿、置顶、系列、系列序号、访问密码、密码提示、正文。

- 草稿：勾选后文章不参与构建
- 封面：上传到媒体库后引用 `/images/posts/<文件名>`，也可直接填写已有图片地址

## 发布流程

保存后 Pages CMS 会直接提交到 `master`，Vercel 自动构建，通常 1–3 分钟后上线。

- 保存即提交，没有独立的预览环境；需要预览时本地运行 `pnpm dev`
- frontmatter 写错（例如缺少必填字段）会导致构建失败，可在 Vercel 部署记录中查看原因
