# Guacamole Server：金蝶旧版控件剪贴板兼容分支

专用分支：`kingdee-compat`。它从 Apache Guacamole Server 1.6.0 开始，
只修改 RDP 剪贴板从浏览器向远程 Windows 公布的文本格式。

## 修改内容

官方 1.6.0 同时公布：

```text
CF_TEXT        -> Guacamole 固定转换为 CP-1252
CF_UNICODETEXT -> UTF-16
```

本分支只公布：

```text
CF_UNICODETEXT -> UTF-16
```

当旧版金蝶控件请求 `CF_TEXT` 时，由远程 Windows 从 Unicode 剪贴板
本地合成 ANSI 文本。对于 ANSI 代码页为 936 的简体中文 Windows，
这可避免中文先被 Guacamole 转成 CP-1252 而变成问号。

补丁不会修改 Guacamole Web、PostgreSQL、用户、连接配置或远程金蝶服务器。
它只影响“浏览器剪贴板 -> RDP 远程 Windows”的纯文本格式公布方式。

## 镜像

GitHub Actions 从 `kingdee-compat` 分支构建并发布：

```text
ghcr.io/lztsgi/guacd-kingdee-compat:kingdee-compat
```

`kingdee-compat` 是随兼容分支更新的固定标签。每次构建还会发布
`sha-xxxxxxx` 标签，便于固定或回退到特定提交。

## Compose 切换

在原 `compose.yml` 中，只替换 `guacd` 服务的 `image`；其余环境变量、
卷和网络保持原样：

```yaml
  guacd:
    image: ghcr.io/lztsgi/guacd-kingdee-compat:kingdee-compat
    container_name: guacd
    hostname: guacd
    restart: unless-stopped

    environment:
      TZ: ${TZ}

    volumes:
      - ${DRIVE_PATH}:/drive

    networks:
      - guacamole-net
```

在 Compose 目录执行：

```bash
cd /root/docker/guacamole
cp compose.yml compose.yml.before-kingdee-compat
docker compose pull guacd
docker compose up -d --no-deps guacd
docker compose ps
docker logs --tail 50 guacd
```

切换 `guacd` 会中断当时正在进行的远程桌面会话，但不会操作 PostgreSQL
容器或 `${DB_PATH}` 数据目录。

测试文本：

```text
测试中文123ABC
```

## 回滚

将 `guacd` 服务恢复为：

```yaml
  guacd:
    image: guacamole/guacd:${GUACAMOLE_VERSION}
```

保留原来的 `container_name`、`hostname`、环境变量、卷和网络，然后执行：

```bash
cd /root/docker/guacamole
docker compose up -d --no-deps guacd
```

如需完全恢复此前的 Compose 文件：

```bash
cp compose.yml.before-kingdee-compat compose.yml
docker compose up -d --no-deps guacd
```

## 适用范围与限制

- 初始基线：Apache Guacamole Server 1.6.0。
- 目标：远程 Windows 的系统 ANSI 代码页为 936，旧应用偏好 `CF_TEXT`。
- 这是一项针对特定兼容问题的定制，不是 Apache 官方补丁。
- `CF_TEXT` 无法覆盖全部 Unicode 字符；本补丁的重点是常用简体中文兼容。
- 修改只影响从浏览器粘贴到远程 Windows。远程复制回浏览器的读取逻辑不变。

## 上游同步

`.github/workflows/sync-upstream-stable.yml` 每周检查 Apache 官方仓库的
最新 1.x 正式版本标签（只接受 `1.x.y`，忽略 RC 和未发布的开发分支）。发现新正式版时，
工作流会将其合并到 `kingdee-compat`，验证 Unicode-only 补丁仍然存在，然后直接
推送到本分支并触发新的 Docker 镜像构建。它不会创建或提交任何上游 PR。

如果官方修改了同一段剪贴板代码并产生冲突，工作流会失败并停止，不会强制覆盖。
也可以在 GitHub 的 Actions 页面手动运行该工作流。

## 上游许可

本仓库沿用 Apache Guacamole Server 的 Apache License 2.0。
