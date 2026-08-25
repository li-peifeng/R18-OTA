# R18-OTA

R18 数据库增量更新包仓库。

Avdb 默认发布仓库：

```text
https://github.com/li-peifeng/R18-OTA
```

增量下载源与发布仓库相互独立：下载源由全局“更新服务器”选择控制，发布仓库
用于生成更新包后创建 GitHub Release。发布仓库必须是当前 GitHub API Token
有发布 Release 资产权限的仓库。

当 Avdb 开启 `允许导出 OTA 包` 时，必须同时配置：

- R18 OTA 发布仓库地址；
- Avdb“网络设置”中的 `GitHub API Token`。

仓库地址和 Token 任一缺失时，Avdb 会阻止保存导出开关，避免生成更新包后才
发现无法发布到 GitHub Release。Token 需要具有向目标仓库发布 Release 资产的权限。

## Release 资产约定

每个包含 R18 更新的 Release 必须上传：

```text
r18-update-manifest.json
```

以及 manifest 中 `asset_name` 指向的数据库增量更新包。manifest 使用
`schema_version: 1`，版本号统一使用 `YYYY-MM-DD`：

```json
{
  "schema_version": 1,
  "product": "avdb-r18",
  "latest_version": "2026-08-25",
  "packages": [
    {
      "version": "2026-08-25",
      "base_version": "2026-08-18",
      "package_type": "delta",
      "asset_name": "r18-delta-2026-08-18-to-2026-08-25.sqlite.gz",
      "inserted_rows": 1200,
      "updated_rows": 800,
      "deleted_rows": 300,
      "change_count": 2300,
      "sha256": "<64 位小写 SHA256>",
      "size": 123456789
    }
  ]
}
```

约束：

- `delta` 必须有 `base_version`，并且基线版本早于目标版本。
- GitHub/Gitee 增量源只发布和使用 `delta` 包；即使 manifest 中存在
  `full` 条目，Avdb 也不会从 Release 下载它。
- `asset_name` 必须与同一个 Release 中的真实资产名称完全一致。
- `sha256` 必须是 64 位小写十六进制字符串。
- Avdb 会沿 `base_version -> version` 解析版本链。
- 版本差异超过 3 个、累计变化超过 30,000 条、版本链断裂或本地没有基线时，
  自动任务停止增量更新并提示手动从官方 `R18.DEV` 下载、导入全量包。
- 全量包只通过 Avdb 页面中的“下载并导入 R18 数据库”或“手动导入 R18 数据库”
  使用，不由自动任务处理。

## Avdb 设置

Avdb 的 R18 设置中有两个相互独立的开关；下载源统一由基础网络设置选择：

- `自动增量更新`：允许客户端定时检测并处理 R18 增量源。
- `允许导出 OTA 包`：允许生成用于本仓库 Release 的更新包，默认关闭。
- `R18 OTA 发布仓库`：导出任务发布更新包的目标仓库。

“更新服务器选择”同时影响资源库增量更新、R18 增量更新和应用内 OTA。打开
`允许导出 OTA 包` 后，导出任务才能创建更新包和 manifest；关闭时不应生成或
发布任何 R18 OTA 资产，发布仓库配置也不显示。
发布仓库地址和 GitHub API Token 都配置完成后，才具备正常发布条件。

## 发布示例

在已经生成并校验 manifest 与更新包后，可以使用 GitHub CLI 创建 Release：

```bash
gh release create r18-2026-08-25 \
  r18-update-manifest.json \
  r18-delta-2026-08-18-to-2026-08-25.sqlite.gz \
  --repo li-peifeng/R18-OTA \
  --title 'R18 数据库 2026-08-25'
```

Release 发布后，Avdb 设置页点击“检查增量源”即可看到本地版本、目标版本、版本跨度、变化条数和增量/手动全量提示。
