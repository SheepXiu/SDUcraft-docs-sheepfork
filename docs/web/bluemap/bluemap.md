author@xhbsh

# BlueMap 部署指南
本文介绍使用 **BlueMap CLI** 渲染并通过 Nginx 对外提供地图网页服务的完整流程。  
更多高级功能请参考 [BlueMap 官方文档](https://bluemap.bluecolored.de/)。

## 1. 环境准备

1. 安装 Java 17+（推荐 Java 21）。
2. 准备一个可运行的 MC 服务端目录（包含世界存档）。
3. 下载 BlueMap CLI：
   [BlueMap CLI Releases](https://github.com/BlueMap-Minecraft/BlueMap/releases)

建议目录结构如下（示例）：

```text
/data/bluemap/
  ├── BlueMap-5.x-cli.jar
  ├── web/                  # BlueMap 前端静态文件
  ├── data/                 # BlueMap 渲染输出数据
  ├── config/               # BlueMap 配置文件
  └── logs/
```

## 2. 首次初始化

进入 BlueMap 目录后执行：

```bash
java -jar BlueMap-5.x-cli.jar
```

首次运行会自动生成 `web/`、`data/`、`config/` 等目录。  
然后编辑 `config/core.conf` 和 `config/maps/` 下的地图配置文件。

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

关键字段：

1. `id`：地图唯一 ID，会出现在网页 URL 中。
2. `name`：网页显示名称。
3. `world`：世界存档绝对路径（必须能被 BlueMap 读取）。

### 3.2 多地图与材质包限制

BlueMap 一次渲染进程只能使用当前加载的一组材质包配置。  
如果不同地图要用不同材质，建议按“**分批渲染 -> 最后合并到一个 web**”处理（见第 5 节）。

## 4. 材质包配置（可选）

如果服务端使用了资源包，建议在渲染时加载相同资源包以避免方块贴图错误。

将资源包放入固定目录后，在配置中声明（示意）：

```hocon
resource-pack: "/path/to/resourcepack.zip"
```

若为多个资源包，可按官方文档顺序配置，后加载的资源会覆盖先加载资源。  
注意：这里的资源包是“本次渲染任务共享”的，不会自动按地图切换。

## 5. 分批渲染并合并到一个 BlueMap Web（推荐）

适用场景：`s4`、`s3`、`fc`、`campus` 等地图需要使用不同材质包。

### 5.1 基本思路

1. 按材质包拆分为多个渲染批次（每批可包含一个或多个地图）。
2. 每批使用独立配置目录与输出目录渲染。
3. 最终把所有批次的地图数据合并到统一 `web/maps/` 下。
4. 手动维护统一 `web/settings.json` 中的 `maps` 列表。

### 5.2 示例目录结构

```text
/data/bluemap/
  ├── batch-a/   # 例如材质包 A（渲染 s4）
  │   ├── config/
  │   ├── web/
  │   └── data/
  ├── batch-b/   # 例如材质包 B（渲染 s3 + fc）
  │   ├── config/
  │   ├── web/
  │   └── data/
  └── merged-web/
      ├── index.html
      ├── settings.json
      └── maps/
```

### 5.3 每个批次分别渲染

在每个 `batch-*` 下：

1. 仅保留该批次要渲染的 `config/maps/*.conf`。
2. 该批次 `core.conf` / `webapp.conf` 中只配置对应材质包。
3. 执行渲染（首次 `-r`，后续 `-u`）。

示例命令（以 `batch-a` 为例）：

```bash
cd /data/bluemap/batch-a
java -jar ../BlueMap-5.x-cli.jar -r
```

### 5.4 合并地图数据

1. 选择一个批次生成的 `web/` 作为前端模板，复制到 `merged-web/`。
2. 将各批次的 `web/maps/<map-id>/` 复制到 `merged-web/maps/`。
3. 确保 `map-id` 唯一，不同批次不要重名。

示例：

```bash
cp -r /data/bluemap/batch-a/web/* /data/bluemap/merged-web/
cp -r /data/bluemap/batch-b/web/maps/* /data/bluemap/merged-web/maps/
```

### 5.5 修改 `web/settings.json`

在 `merged-web/settings.json` 中，把所有地图 ID 加入 `maps` 数组，例如：

```json
{
  "maps": ["s4", "s3", "fc", "campus"]
}
```

并在用于维护 `merged-web` 的 BlueMap 配置中设置：

```hocon
update-settings-file: false
```

这样可以避免后续渲染覆盖你手动合并后的 `settings.json`。

## 6. 单批次渲染命令速查

### 6.1 全量渲染

```bash
java -jar BlueMap-5.x-cli.jar -r
```

### 6.2 增量渲染（推荐日常使用）

```bash
java -jar BlueMap-5.x-cli.jar -u
```

渲染完成后，地图数据会输出到 `data/` 目录。

## 7. 通过 Nginx 反向代理发布

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

## 8. 常见问题排查

### 8.1 网页空白或 404

1. 检查 BlueMap 进程是否在监听对应端口（`8100~8103`）。
2. 检查 Nginx `location` 路径结尾 `/` 与 `proxy_pass` 结尾 `/` 是否匹配。
3. 检查防火墙与安全组策略。

### 8.2 方块贴图错误

1. 检查材质包路径是否正确。
2. 检查渲染时是否加载了与服务端一致的资源包。
3. 重新执行一次全量渲染 `-r`。

### 8.3 渲染速度慢

1. 首次全量渲染耗时长属正常。
2. 日常更新请使用增量渲染 `-u`。
3. 可将渲染任务放到低峰期定时执行。

### 8.4 合并后地图不显示

1. 检查 `merged-web/maps/<map-id>/` 目录是否存在且可访问。
2. 检查 `merged-web/settings.json` 的 `maps` 数组是否包含该 `map-id`。
3. 检查 `map-id` 与地图配置文件名（去掉 `.conf` 后）是否一致。
