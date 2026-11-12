# moeflow-deploy

用于自己部署萌翻服务的配置。

请参见部署视频演示 https://www.bilibili.com/video/BV1EX4y1J7MA/

## docker compose

建议的部署方式。

0. 安装 [docker](https://docs.docker.com/engine/install/) 和 [docker-compose-plugin](https://docs.docker.com/compose/install/)
    - 注: 旧版本docker-compose可能无法正确设置环境变量. docker-compose-plugin v2.27.0 经测试可用.
1. 下载本repo的 [最新release](https://github.com/moeflow-com/moeflow-deploy/releases) 并解压
    - 非开发者不建议clone本repo: 代码最新状态可能和image不匹配
2. 将 `.env-backend.sample` 模板复制为 `.env-backend` ，并按模板内说明设置环境变量
3. 将 `.env.sample` 模板复制为 `.env` ，并按模板内说明设置环境变量
4. 运行 `docker compose up`

## FAQ

### 修改主页内容

升级前端版本至 `v1.0.3` 后端版本至 `v1.0.8` 后，即可在管理员页面-站点管理中，设置主页 HTML 和 CSS，以下为示例：

主页 HTML

```html
<div class="homepage-title">
  <h1>欢迎使用萌翻！</h1>
  <p>
    <a href="login">登陆</a>
  </p>
  <p>
    <a href="register">注册</a>
  </p>
</div>
<div class="homepage-footer">备案号</div>
```

主页 CSS

```css
html {
  width: 100%;
  height: 100%;
}
body {
  width: 100%;
  height: 100%;
  margin: 0;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}
.homepage-title {
  flex: auto;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}
.homepage-footer {
  height: 50px;
  text-align: center;
}
```

## C# 后端群组（重构版）

`docker-compose.csharp.yml` 是 C# 后端的部署群组，**与上面的 Python 群组共用同一个 MongoDB**，不新建数据库。它加入 Python 群组创建的外部网络（默认 `moeflow-deploy_default`），因此两个版本可以同时跑、对着同一份数据做对比。

### 组成

| 服务 | 作用 | 端口 |
|---|---|---|
| `moeflow-csharp-api` | HTTP + 任务消费 + 迁移闸门（单容器） | `127.0.0.1:5091` → 8080 |
| `moeflow-csharp-frontend` | nginx，托管 SPA 并反代 `/api/` | `127.0.0.1:5090` → 80 |

默认只有一个后端容器：HTTP 和任务队列在同一个进程里。与 Python 群组相比少了 RabbitMQ 和三个 Celery 容器——任务队列落在 MongoDB 的 `csharp_job` 集合里。

确实需要把消费者拆出来单独扩缩时，才启用可选的 split profile：

```bash
JOBS_WORKER_ENABLED=false docker compose -f docker-compose.csharp.yml --env-file .env-csharp --profile split up -d
```

2 核机器上不建议拆：多一个容器只多约 63 MiB，换不到任何吞吐。

### 部署

1. 复制模板并按注释填写：

   ```bash
   cp .env-csharp.sample .env-csharp
   ```

2. 校验变量（缺项会直接报错，不会等到启动才崩）：

   ```bash
   docker compose -f docker-compose.csharp.yml --env-file .env-csharp config
   ```

3. 启动：

   ```bash
   docker compose -f docker-compose.csharp.yml --env-file .env-csharp up -d
   ```

4. 关注启动日志，确认迁移闸门通过：

   ```bash
   docker compose -f docker-compose.csharp.yml --env-file .env-csharp logs -f moeflow-csharp-api
   # 期望：Migration check clean: no pending migrations
   ```

### 关键约束

- **`AUTH_SECRET` 必须等于旧版的 `SECRET_KEY`**，否则所有存量会话失效（返回 `BadTokenError 1004`）。
- `MIGRATIONS_MODE=check` 表示有待执行迁移时 `/health/ready` 返回 503 而不是带病服务；首次部署或升级迁移时临时改成 `apply`，跑完再切回 `check`。
- `NOTIFICATIONS_WORKER_ENABLED` 和 `STORAGE_POOL_WORKER_ENABLED` **默认关闭**：它们会真的发邮件、真的搬对象。确认目标数据集后再打开。
- 通知邮件在 465 端口需要 `MAIL_SECURE_SOCKET_OPTIONS=SslOnConnect`，否则会按旧版语义强制 STARTTLS 而失败。
- 上传上限由 nginx 的 `MAX_CONTENT_LENGTH_MB` 和后端自带的 1 GiB 限制共同决定，前者要大于等于后者。

### 兼容旧变量

C# 后端内置了旧变量名到新键名的映射（`SECRET_KEY`→`Auth:Secret`、`EMAIL_*`→`Mail:*`、`OSS_*`→`Storage:*`、`MONGODB_*`→`Mongo:Uri` 等），显式的新键优先。因此迁移期可以直接沿用旧的 `.env-backend`，不必一次改完。
