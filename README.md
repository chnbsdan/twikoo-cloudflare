# Twikoo 评论系统的 Cloudflare 部署

这是 [Twikoo](https://twikoo.js.org/zh/intro.html) 评论系统的 Cloudflare 部署版本。相比 Vercel/Netlify + MongoDB 等其他部署方式，它大幅改善了冷启动延迟（从 `6s` 降至 `<0.5s`）。延迟的改善主要得益于对 Cloudflare Workers 的巨大优化，以及 HTTP 服务器和数据库（Cloudflare D1）之间的集成环境。

## 部署步骤

1. 安装 npm 包：
   ```shell
   npm install
   ```

2. 由于 Cloudflare Workers 的免费套餐对打包体积有严格的 1MiB 限制，我们需要手动删除一些包以保持打包体积在限制内。这些包由于 Cloudflare Workers 的 Node.js [兼容性问题](#已知限制)，无法使用。
   ```shell
   echo "" > node_modules/jsdom/lib/api.js
   echo "" > node_modules/tencentcloud-sdk-nodejs/tencentcloud/index.js
   echo "" > node_modules/nodemailer/lib/nodemailer.js
   ```

3. 登录你的 Cloudflare 账号：
   ```shell
   npx wrangler login
   ```

4. 创建 Cloudflare D1 数据库并设置数据表结构：
   ```shell
   npx wrangler d1 create twikoo
   ```

5. 从上一步的输出中复制 `database_name` 和 `database_id` 两行内容，粘贴到 `wrangler.toml` 文件中，替换原有的值。

6. 设置 Cloudflare D1 数据表结构：
   ```shell
   npx wrangler d1 execute twikoo --remote --file=./schema.sql
   ```

7. 创建 Cloudflare R2 存储桶：
   ```shell
   npx wrangler r2 bucket create twikoo
   ```

8. 将 R2 的域名更新到 `wrangler.toml` 文件中，替换 `R2_PUBLIC_URL` 的值。

9. 部署 Cloudflare Worker：
   ```shell
   npx wrangler deploy --minify
   ```

10. 如果一切顺利，你会在命令行中看到类似这样的地址：`https://twikoo-cloudflare.<你的用户名>.workers.dev`。你可以访问该地址。如果一切都设置正确，你会在浏览器中看到类似这样的一行：
    ```
    {"code":100,"message":"Twikoo 云函数运行正常，请参考 https://twikoo.js.org/frontend.html 完成前端的配置","version":"1.6.33"}
    ```

11. 当你设置前端时，第 6 步中的地址（包括 `https://` 前缀）应作为 `twikoo.init` 中的 `envId` 字段使用。

> 自动部署：[查看博客](https://blog.mingy.org/2024/12/hexo-add-twikoo/)

## 已知限制

由于 Cloudflare Workers 仅与 Node.js [部分兼容](https://developers.cloudflare.com/workers/runtime-apis/nodejs/)，Twikoo 的 Cloudflare 部署存在以下功能限制：

1. 不能使用环境变量（`process.env.XXX`）来控制应用行为。
2. 无法集成腾讯云。
3. 无法基于 IP 地址查找地理位置（`@imaegoo/node-ip2region` 包的兼容性问题）。
4. 由于 `jsdom` 包的兼容性问题，不能使用 `dompurify` 来过滤评论内容。我们改用 [`xss`](https://www.npmjs.com/package/xss) 包进行 XSS 过滤。
5. 在此部署中，我们不会规范 `/some/path/` 和 `/some/path` 之间的 URL 路径。这是因为编写 Cloudflare D1 SQL 查询来统一这两种路径不太容易。如果你的网站同一页面可以有带或不带尾部斜杠的路径，你可以在 `twikoo.init` 中显式设置 `path` 字段。
6. 图片上传使用了 Cloudflare R2 存储。
7. 由于使用了 [axios-cf-worker](https://github.com/wuzhengmao/axios-cf-worker)，`pushoo.js` 可以正常工作。

## 配置邮件通知

由于 `nodemailer` 包的兼容性问题，通过 SMTP 发送通知邮件的邮件集成方式无法直接使用。在此 Worker 中，我们通过 SendGrid 的 HTTPS API 支持邮件通知。要启用 SendGrid 邮件集成，请按照以下步骤操作：

1. 确保你有一个可用的 SendGrid 账号（SendGrid 提供免费套餐，每天可发送最多 100 封邮件）或 MailChannels 账号（每月免费 3000 封邮件），并创建一个 API 密钥。
2. 在配置中设置以下字段：
   - `SENDER_EMAIL`：发件人的邮箱地址。需要在 SendGrid 中验证。
   - `SENDER_NAME`：发件人显示的名称。
   - `SMTP_SERVICE`：`SendGrid`。
   - `SMTP_USER`：填写任意非空值。
   - `SMTP_PASS`：API 密钥。
3. 可选地，你可以设置其他配置值来自定义通知邮件的样式。
4. 在配置页面中，点击「发送测试邮件」按钮，确保集成工作正常。
5. 在你的邮件服务商中，确保收到的邮件不会被归类为垃圾邮件。

---

如果你遇到任何问题，或对此部署有任何疑问，可以发送邮件至 tao@vanjs.org。

