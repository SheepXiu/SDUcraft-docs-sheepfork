author@xhbsh

# BlueMap 部署指南
本文介绍使用 **BlueMap CLI** 渲染并通过 Nginx 对外提供地图网页服务的完整流程。  
更多高级功能请参考 [BlueMap 官方文档](https://bluemap.bluecolored.de/)。

## 1. 环境准备

1. 安装 Java 17+（推荐 Java 21）。
2. 准备一个可运行的 MC 服务端目录（包含世界存档）。
3. 下载 BlueMap CLI：
   [BlueMap CLI Releases](https://github.com/BlueMap-Minecraft/BlueMap/releases)


## 2. 首次初始化

进入 BlueMap 目录后执行：

```bash
java -jar BlueMap-5.x-cli.jar
```

首次运行会自动生成 `web/`、`data/`、`config/` 等目录。  
然后编辑 `config/core.conf` 和 `config/maps/` 下的地图配置文件。
目录结构如下：

```text
/data/bluemap/
  ├── BlueMap-5.x-cli.jar
  ├── web/                  # BlueMap 前端静态文件
  ├── data/                 # BlueMap 渲染输出数据
  ├── config/               # BlueMap 配置文件
  └── logs/
```

## 3. 配置地图

### 3.1 指定世界目录

在 `config/maps/` 中新建或修改地图配置，例如 `s4.conf`：

```hocon
map {
  id: "s4"
  name: "Survival-4"
  world: "/path/to/minecraft/world"
  sky-color: "#7dabff"
}
```
1. `id`：地图唯一 ID，会出现在网页 URL 中。
2. `name`：网页显示名称。
3. `world`：世界存档绝对路径（必须能被 BlueMap 读取）。

### 3.2 多地图与材质包限制

BlueMap 一次渲染进程只能使用当前加载的一组材质包配置。  
如果不同地图要用不同材质，建议按“**临时挪走其他地图配置 -> 只渲染目标地图 -> 再放回配置**”处理（见第 5 节）。

## 4. 材质包配置

将资源包放入`config/packs`：
若为多个资源包，可按官方文档顺序配置，后加载的资源会覆盖先加载资源。  
> 注意：这里的资源包是“本次渲染任务共享”的，不会自动按地图切换。

## 5. 同一套目录下按地图分次渲染（推荐）

适用场景：不同地图需要不同材质包，但你不想额外开分支目录。

### 5.1 基本思路

1. `config/maps/` 里只保留本次要渲染的地图配置。
2. 其余地图配置临时挪走（例如放到 `config/maps_hold/`）。
3. 按本次材质包配置执行渲染。
4. 渲染完成后，把挪走的配置再放回 `config/maps/`。

这样 BlueMap 只会更新当前保留的地图，不会动前面已经渲染好的其他地图。

### 5.2 操作步骤

示例：本次只渲染 `campus`，其余先挪走。

```bash
cd /data/bluemap
mkdir -p config/maps_hold

# 挪走不需要渲染的地图配置（示例）
mv config/maps/s4.conf config/maps/s3.conf config/maps/fc.conf config/maps_hold/

# 执行渲染（首次建议 -r，日常可用 -u）
java -jar BlueMap-5.x-cli.jar -r

# 渲染完成后放回配置
mv config/maps_hold/*.conf config/maps/
```

> 注意：不要删除配置，建议只做“移动-渲染-移回”。

### 5.3 `web/settings.json` 维护

如果你是通过手工方式维护 `web/settings.json` 的 `maps` 列表，建议在配置里设置：

```hocon
update-settings-file: false
```

避免后续渲染自动覆盖你手工调整的内容。

更新官网上的 BlueMap 后重启 BlueMap CLI 即可。

```bash
systemctl restart bluemap.service
```

## 6. 通过 Nginx 反向代理发布

BlueMap CLI 可以启动内置 Web 服务（默认监听本机端口），再由 Nginx 转发。  
示例（与现网配置一致）：

```nginx
location /bluemap/campus/ {
    proxy_pass http://127.0.0.1:8103/;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

配置完成后执行：

```bash
nginx -t && systemctl reload nginx
```

## 7. 常见问题排查

::: qa 方块贴图错误
1. 检查材质包路径是否正确。
2. 检查渲染时是否加载了与服务端一致的资源包。
3. 重新执行一次全量渲染 `-r`。
:::

::: qa 挪走配置后地图缺失或未更新
1. 检查渲染完成后是否已把 `config/maps_hold/*.conf` 全部移回 `config/maps/`。
2. 检查本次渲染时保留的 `config/maps/*.conf` 是否就是目标地图。
3. 若你手工维护了 `web/settings.json`，检查其中 `maps` 数组是否仍包含全部地图 ID。
:::
