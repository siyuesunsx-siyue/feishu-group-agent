---
name: feishu-group-agent
description: >-
  Put a coding agent into a Feishu (Lark) group so members can @ it and get
  answers, including a 24-hour setup that keeps replying after the operator
  closes their computer. Use when the user asks to add an agent or bot to a
  飞书群, connect lark-channel-bridge, enable group Q&A, subscribe to
  im.message.receive_v1, or make a Feishu bot reply without a local chat
  staying open.
---

# 让 Agent 进入飞书群并回答问题

目标：飞书群成员 @ 机器人后得到回答。人关掉自己的电脑后仍然要答，就把监听放在一台一直开着的机器上。

## 先选运行方式

24 小时回答用常驻机器上的 [`lark-channel-bridge`](https://github.com/zarazhangrui/lark-coding-agent-bridge)。它握着飞书长连接，收到 @ 后在那台机器上启动 Agent，再把回复发回群里。

当前电脑上的 `lark-cli event consume` 只在这次对话开着时有用。电脑或对话一关就停。隔一段时间拉取聊天记录也不是即时回答。

一个飞书应用只有一条实用的监听线。同一个应用不能一边在个人电脑听、一边在常驻机器听。飞书可以给一个应用建多条长连接，但每条事件只随机投递给其中一条。不同 Agent 要用不同应用，各自一条线。

进了群不等于占着监听线。占线的是正在跑的 bridge 或 `event consume`。

## 开工前确认

- 用现有飞书应用，还是新建。用户没明确要求新建时，不要再造一个应用去占同一条线。
- 目标群的准确名字，以及它是内部群还是外部群。
- Agent 跑在哪台常驻机器上。那台机器必须能访问这个 Agent 的模型服务，否则日志里已经「已连接」，回复仍会卡住。
- 谁是应用所有者。群里其他人要得到回复，所有者得在那个群里发一次 `/invite group`。默认只有所有者的消息会被回答，其他人 @ 了也没有回复。

## 24 小时路径

错误码、换密码、两份 Secret 见 [reference.md](reference.md)。

1. 在[飞书开放平台](https://open.feishu.cn/app)准备应用。打开机器人能力。事件订阅方式选「使用长连接接收事件」，订阅 `im.message.receive_v1`（接收消息）。这两项都要出现在已发布版本里。外部群还要发布「对外共享」。凭证、事件、版本页分别是 `https://open.feishu.cn/app/<APP_ID>/baseinfo`、`/event`、`/version`。
2. 常驻机器安装 Node.js 20.12 或更高、`lark-cli`、要跑的 Agent，并完成该 Agent 自己的登录。安装桥：`npm i -g lark-channel-bridge`。服务命令不要用 `npx`，缓存清掉后后台服务会找不到程序。
3. 用已有应用初始化，不要让首次向导再创建一个应用：`lark-channel-bridge run --app-id <APP_ID> --profile <名字> --agent <claude|codex>`。已安装的桥如果在 `--help` 里还列出别的 agent 种类，用那个种类。App Secret 只从提示或标准输入进入。
4. 前台日志出现已连接后，停掉前台，改成系统服务：`lark-channel-bridge start --profile <名字>`。Linux 上先执行 `sudo loginctl enable-linger <登录用户>`，否则 SSH 断开后用户服务会停。
5. 停掉其他机器上这个 App ID 的监听：`lark-cli event stop --app-id <APP_ID> --force`。先用这个命令，不要先 `kill -9`。用户自己的飞书客户端不是这条监听线，不要结束它。
6. 用已经在群里的用户身份把机器人加进去。先看 `lark-cli im chat.members create --help`。已跑通的形状是 `--as user`，`member_id_type` 为 `app_id`，`id_list` 里放 App ID，`succeed_type` 为 `1`。
7. 应用所有者在目标群里 @ 机器人，发送 `/invite group`。
8. 请用户在目标群 @ 机器人说一句话。处理卡片会先出现。完整回复要等 Agent 跑完，常见要几分钟。日志里 `queue=` 接近 `0ms`，表示消息已经即时进桥，慢在 Agent 本身。

群里的 Agent 还要自己调飞书接口时，按 reference 里的「两份密码」再写一份。`secrets set` 只更新监听用的那一份。

## 临时调试

只在用户接受「这台电脑和这次对话必须开着，常驻服务会失声」时使用：

```bash
lark-cli event consume im.message.receive_v1 --as bot
```

结束时执行 `lark-cli event stop --app-id <APP_ID> --force`，再启动常驻服务。不要为了查看在线连接数而在个人电脑上再开一条 `event consume`。

## 完成标准

- 常驻服务是 active。本次启动之后的日志有「已连接」和「正在监听消息」。
- 没有第二台机器占着同一个 App ID。
- 目标群成员里能看到这个机器人。
- 所有者之外的人要提问时，群里已经发过 `/invite group`。
- 有人 @ 之后，群里先有处理卡片，然后有回复。

## 不要做

- 不要把 App Secret 写进仓库、命令参数、日志或群聊，也不要在回复里复述。
- 不要为了同一条监听线新建第二个应用。
- 不要把轮询聊天记录当成 24 小时方案。
- 不要在个人电脑和常驻机器同时听同一个应用。
- 不要假设群里的 Agent 看得到操作者电脑上的文件或 `127.0.0.1`。文件放到它的工作目录，或发到群里。
