

## 代码评审 Agent 

- 目标：把“代码评审”做成产品闭环——提交代码或 git diff → 结构化评审 → 将评审结果作为上下文继续对话，产出可执行修改方案与关键代码片段 
- 部署：后端 Spring Boot（systemd）+ 前端静态页面（Nginx 静态），GitHub Actions 产物构建 

## 架构概览 

- 前端（静态页面） 
  - 页面：Code Review（评审代码/评审 diff）+ Chat（跟进对话） 
  - 静态托管：Nginx，反代 `/api/*`、`/mcp/*` 到后端 
  - 入口与样式：`openai-code-review-web/src/App.jsx`、`openai-code-review-web/src/styles.css` 

- 后端（Spring Boot） 
  - 接口： 
    - `POST /api/code-review/code`：评审代码文本 
    - `POST /api/code-review/diff`：评审 git diff 
    - `POST /api/chat`：带上下文对话（将评审结果+代码/变更拼接进 prompt） 
    - `GET /mcp/tools` / `POST /mcp/call`：MCP 工具化接口 
  - 配置与依赖注入：`openai-code-review-test/src/main/java/.../AgentConfiguration.java` 

- SDK（轻量 Agent / Tool / Chain / MCP） 
  - Agent 核心：最小 tool-calling 循环（采用 ReAct 模式实现多步推理循环，模型输出约定块触发工具执行，记录 scratchpad） 
  - Chat 模型适配（ChatGLM HTTP 调用） 
  - MCP：工具注册对外暴露（评审工具、RAG 等可扩展） 

- 部署（可复现） 
  - GitHub Actions 构建产物（后端 jar + 前端 dist） 
  - 服务器：systemd 常驻后端 + Nginx 前端静态与反代 
  - 一键脚本：支持清空旧部署、自动解压产物、发布 

## 技术栈 

- 前端：静态页面 
- 后端：Spring Boot（JDK 11+） 
- Agent/SDK：Java 轻量实现（参考 langchain4j 思想，但未直接依赖），采用 ReAct 模式 
- 大模型：ChatGLM（HTTP API） 
- 基础设施：Nginx（静态+反代）、systemd（后端常驻） 
- CI/CD：GitHub Actions（Artifacts 构建并下载） 

模块分层 
- 前端：静态页面，两个面板（Code Review、Chat），静态资源由 Nginx托管 
  - 页面入口： App.jsx 
  - 样式与响应式布局： styles.css 
  - 前端接口封装： api.js 
- 后端：Spring Boot（REST API），聚合 SDK 能力，服务端口默认 8091 
  - 启动入口： Application.java 
  - API 控制器（评审/聊聊/MCP）： CodeReviewController 、 ChatController 、 McpController 
  - 业务服务（评审核心）： CodeReviewService 
  - 依赖注入与工具注册： AgentConfiguration 
- SDK：轻量 Agent / Chain / RAG / MCP 抽象，统一模型通道 
  - Agent核心（tool-calling，ReAct模式）： SimpleAgent 
  - ChatModel适配（ChatGLM）： ChatGlmChatModel 
  - Chain（最小链式处理）： LinearChain 
  - MCP 数据结构与工具注册：见 McpController 与 ToolRegistry 
## 功能点 

- 评审代码文本：生成结构化 Markdown（评分、问题、建议、修改示例） 
- 评审 git diff：聚焦改动本身给出评审 
- 跟进怎么改：把“代码/或diff + 评审结果”打包成上下文后在 Chat 持续推理 
- MCP 工具化：`/mcp/tools` 列工具，`/mcp/call` 调用工具（方便能力中心化） 

## 本地运行（开发联调） 

- 后端 
  ```bash 
  # 需要 JDK 11+、Maven 3.8+ 
  mvn -DskipTests package 
  java -jar openai-code-review-test/target/openai-code-review-test.jar 
  ``` 
  环境变量（或 application-dev.yml）： 
  - `CHATGLM_APIHOST= `https://open.bigmodel.cn/api/paas/v4/chat/completions` ` 
  - `CHATGLM_APIKEYSECRET=YOUR_KEY_SECRET` 
  - `AGENT_RAG_STORE_PATH=data/rag.jsonl`（可选） 

- 前端 
  ```bash 
  cd openai-code-review-web 
  npm i 
  npm run dev 
  ``` 
  开发代理：`/api`、`/mcp` → `http://localhost:8091` 

## 部署流程（推荐：Actions 构建产物 + 服务器部署） 

- 在 GitHub Actions 运行 “Build Demo Artifacts” 并下载两个压缩包： 
  - `openai-code-review-backend-jar.zip`（内含 `openai-code-review-test.jar`） 
  - `openai-code-review-web-dist.zip`（内含 `dist/`） 

- 服务器部署后端（systemd） 
  ```bash 
  sudo mkdir -p /opt/agent/data 
  sudo cp openai-code-review-test.jar /opt/agent/agent.jar 

  sudo tee /opt/agent/.env >/dev/null << 'EOF' 
  CHATGLM_APIHOST= `https://open.bigmodel.cn/api/paas/v4/chat/completions` 
  CHATGLM_APIKEYSECRET=YOUR_KEY_SECRET 
  AGENT_RAG_STORE_PATH=/opt/agent/data/rag.jsonl 
  EOF 
  sudo chmod 600 /opt/agent/.env 

  sudo tee /etc/systemd/system/agent.service >/dev/null << 'EOF' 
  [Unit] 
  Description=Lightweight Agent Backend 
  After=network.target 

  [Service] 
  Type=simple 
  WorkingDirectory=/opt/agent 
  EnvironmentFile=/opt/agent/.env 
  ExecStart=/usr/bin/java -jar /opt/agent/agent.jar --server.port=8091 --agent.ragStorePath=/opt/agent/data/rag.jsonl 
  Restart=always 
  RestartSec=3 

  [Install] 
  WantedBy=multi-user.target 
  EOF 

  sudo systemctl daemon-reload 
  sudo systemctl enable agent 
  sudo systemctl restart agent 
  sudo systemctl status agent --no-pager 
  ``` 

- 服务器部署前端（Nginx 静态 + 反代） 
  ```bash 
  sudo mkdir -p /var/www/agent-ui 
  sudo rm -rf /var/www/agent-ui/* 
  sudo cp -r dist/* /var/www/agent-ui/ 

  sudo tee /etc/nginx/conf.d/agent.conf >/dev/null << 'EOF' 
  server { 
    listen 80 default_server; 
    server_name _; 

    root /var/www/agent-ui; 
    index index.html; 

    location / { 
      try_files $uri $uri/ /index.html; 
    } 

    location /api/ { 
      proxy_pass http://127.0.0.1:8091; 
    } 
    location /mcp/ { 
      proxy_pass http://127.0.0.1:8091; 
    } 
  } 
  EOF 

  sudo rm -f /etc/nginx/default.d/welcome.conf /etc/nginx/conf.d/welcome.conf 
  sudo nginx -t 
  sudo systemctl enable nginx 
  sudo systemctl restart nginx 
  sudo systemctl status nginx --no-pager 
  ``` 

- 验收 
  ```bash 
  curl -i http://127.0.0.1/mcp/tools      # 期望 200 
  curl -i http://127.0.0.1/api/code-review/code 
  ``` 
  浏览器访问：`http://<服务器IP>/` 

## 交互演示（闭环） 

- Code Review 面板：粘贴代码 → “评审代码”；粘贴 git diff → “评审 diff” 
- 评审结果出来 → “跟进怎么改” 
- Chat 输入：按评审意见逐条给我修改方案，并给出关键修改代码片段 

## 后期优化方向 

- Agent 交互（ReAct 模式） 
  - 提示词优化：减少工具调用占位符输出、强化直接给出修改方案与片段 
  - 工具解析健壮性：LLM 非标准输出的兼容兜底 
  - 多轮裁剪：上下文摘要/截断，基于 token 预算管理 

- 能力扩展 
  - PR 评审入口：支持直接输入 GitHub PR 链接拉取 diff 
  - 服务器仓库评审入口：UI 增加 repoPath/fromRef/toRef 表单触发 `/api/code-review/repo` 
  - RAG 完善：更好的分块、向量化与持久化（换向量库等） 

- 工程化 
  - 容器化：backend（openjdk:11-jre）+ frontend（nginx）→ docker-compose 编排 
  - 监控与日志：标准化访问日志、异常捕获、指标（QPS/RT/错误率） 
  - 安全与鉴权：后端接口 token 校验、Nginx 限流/防护 

