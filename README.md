# clashforme

[CFnew 精简内核](https://github.com/byJoey/cfnew) 的自建镜像。

内核只负责**代理转发**；配置界面、优选列表处理和订阅生成都已剥离到[独立面板](https://cfnew.xy.kg)（源码 [cfnew-panel](https://github.com/byJoey/cfnew-panel)）。

| | |
| --- | --- |
| 上游项目 | https://github.com/byJoey/cfnew |
| 配套面板 | https://cfnew.xy.kg |
| 当前内核版本 | 见 [`VERSION.txt`](VERSION.txt) |
| 完整变量表 | [`docs/variables.md`](docs/variables.md) |
| 上游完整说明 | [`docs/upstream-README.md`](docs/upstream-README.md) |
| 单版本发行说明 | [`docs/releases/`](docs/releases/) |

## 仓库内容

| 路径 | 来源 | 说明 |
| --- | --- | --- |
| `_worker.js` | 上游 | 代理内核本体，唯一需要部署的文件 |
| `pages.zip` | 上游 | Cloudflare Pages 直接上传包，包根目录只有 `_worker.js` |
| `docs/` | 上游 | 变量表、发行说明 |
| `scripts/package-release.py` | 上游 | 校验内核与部署包一致性，并生成发行资产与校验值 |
| `VERSION.txt` | 自动维护 | 当前内核版本号 |
| `.upstream-sync` | 自动维护 | 已同步到的上游 commit |
| `.gitattributes` | 本仓库 | 禁止内核被行尾转换，保证与上游逐字节一致 |
| `.github/workflows/sync-from-cfnew.yml` | 本仓库 | 每 6 小时自动同步一次上游 |

## 自动同步

`.github/workflows/sync-from-cfnew.yml` 每 6 小时检查一次上游 `core-only` 分支，有更新就同步、校验并推送。

**为什么不用 GitHub 原生的 "Sync fork"：** 上游 `core-only` 是重写过历史后的独立根提交，与本仓库没有任何共同祖先，合并会被 git 直接拒绝：

```
$ git merge upstream/core-only
fatal: refusing to merge unrelated histories
```

所以这里不做 merge，改为**按文件取用上游内容**：内核、部署包、文档、脚本都直接从上游分支检出，`docs/`、`scripts/` 整目录替换，因此上游删掉的文件也会同步删除。

同步前会做两项校验，任一不通过就中止、不会提交：

1. `pages.zip` 内的 `_worker.js` 必须与仓库内核逐字节一致
2. 内核与部署包必须与上游分支逐字节一致

同步记录写在 `.upstream-sync`，只记录上游 commit，因此**上游未发版的提交也能同步进来**，不必等 Release。

> 注意：同步只覆盖上游管理的文件，本仓库自己的 `.gitattributes` 和本 README 不会被覆盖。

本地想手动同步一次：

```bash
git remote add upstream https://github.com/byJoey/cfnew.git
git fetch --depth=1 upstream core-only:refs/remotes/upstream/core-only
git checkout refs/remotes/upstream/core-only -- _worker.js pages.zip .gitignore
rm -rf docs scripts
git checkout refs/remotes/upstream/core-only -- docs scripts
mkdir -p docs && git show refs/remotes/upstream/core-only:README.md > docs/upstream-README.md
```

## 部署

内核零依赖、无法本地运行（依赖 `cloudflare:sockets` 与 `WebSocketPair`），只能部署到 Cloudflare。

**Cloudflare Pages**

1. 上传 [`pages.zip`](pages.zip)（包根目录只有 `_worker.js`，整包上传，不要解压后再传文件夹）
2. 变量和机密 → 生产环境 → 文本变量 `u`，值填**你自己的 UUID**（大写 `U` 也支持）
3. 需要在线保存配置时，创建 Workers KV 命名空间，绑定名称填 `C` 或 `c`
4. 兼容日期设为 `2026-01-20`
5. 改完变量、绑定或兼容日期后**重新部署**

**Cloudflare Workers**

1. 新建 Worker 并完成初始部署，进入代码编辑器
2. 把 [`_worker.js`](_worker.js) 内容完整粘贴进去，**ES Module 格式**，保留 `import` 与 `export default`
3. 添加变量 `u`，需要在线保存配置再添加名为 `C` 的 KV 绑定
4. 兼容日期设为 `2026-01-20`，重新部署

部署完成后访问首页显示 `hello` 即正常。管理页面和订阅不在 Worker 上，请到[独立面板](https://cfnew.xy.kg)填 Worker 域名和同一个 UUID 登录。

## ⚠️ 安全提醒

`_worker.js` 里**硬编码了一个默认 UUID** `351c9981-04b6-4103-aa4b-864aa9c91469`。只有当环境变量 `u` / `U` 未设置时才会回退到它。

这个值是公开的，任何人都能查到。**部署前必须设置自己的 `u`**，否则你的代理入口对所有人开放，UUID 也等于公开状态。

配置优先级是 **KV 已保存的值 → 环境变量 → 内置默认值**。在面板保存过某项之后，改同名环境变量不会生效，需要在面板里重置或清掉对应 KV 字段。
