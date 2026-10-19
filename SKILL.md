---
name: feishu-group-agent
description: >-
  Put a coding agent into a Feishu (Lark) group so members can @ it and get
  answers. 24-hour uptime is optional and requires the user to confirm plus an
  always-on machine. Use when the user asks to add an agent or bot to a 飞书群,
  connect lark-channel-bridge, enable group Q&A, subscribe to
  im.message.receive_v1, or keep a Feishu bot replying.
---

# 让 Agent 进入飞书群并回答问题

目标：飞书群成员 @ 机器人后得到回答。先做成「这台电脑开着就能答」。24 小时在线是可选项，必须先向用户确认。

## 开工前确认

- 用现有飞书应用，还是新建。用户没明确要求新建时，不要再造一个应用去占同一条线。
- 目标群的准确名字，以及它是内部群还是外部群。
- 谁是应用所有者。群里其他人要得到回复，所有者得在那个群里发一次 `/invite group`。默认只有所有者的消息会被回答。
- 要不要 24 小时在线。用户没明确说要之前，先说明下面的条件，等用户确认再做。

## 24 小时在线：先确认，再动手

向用户说明这些条件，得到明确同意后再做「24 小时路径」：

- 需要一台一直开着、一直联网的机器。可以是一台 VPS，也可以是一台不关机的电脑。这台机器关掉或断网，群里就不再回答。
- 这台机器必须能访问该 Agent 的模型服务。访问不了时，飞书这边可以显示已连接，回复仍会卡住。
- 机器上要装 Node.js 20.12 或更高、`lark-cli`、Agent 自己的命令行并完成登录，以及全局安装的 `lark-channel-bridge`。
- Linux 上要开 linger，用户退出登录后服务才继续跑。
- 一个飞书应用只有一条实用的监听线。24 小时服务跑起来之后，个人电脑上不能再对同一个应用执行 `event consume`。

用户不需要 24 小时，或没有这样的机器：用下面的「当前电脑回答」。进程或电脑关掉，回答就停。

## 准备应用并进群

错误码见 [reference.md](reference.md)。

1. 在[飞书开放平台](https://open.feishu.cn/app)打开机器人能力。事件订阅方式选「使用长连接接收事件」，订阅 `im.message.receive_v1`（接收消息）。这两项都要出现在已发布版本里。外部群还要发布「对外共享」。凭证、事件、版本页分别是 `https://open.feishu.cn/app/<APP_ID>/baseinfo`、`/event`、`/version`。
2. 用已经在群里的用户身份把机器人加进去。先看 `lark-cli im chat.members create --help`。已跑通的形状是 `--as user`，`member_id_type` 为 `app_id`，`id_list` 里放 App ID，`succeed_type` 为 `1`。
3. 应用所有者在目标群里 @ 机器人，发送 `/invite group`。
4. 请用户在目标群 @ 机器人说一句话，确认有回复。

一个飞书应用只有一条实用的监听线。飞书可以给一个应用建多条长连接，但每条事件只随机投递给其中一条。不同 Agent 要用不同应用。进了群不等于占着监听线，占线的是正在跑的 bridge 或 `event consume`。

## 当前电脑回答

在准备回答的这台电脑上安装 `lark-cli`，用该应用的 App ID 和 App Secret 完成配置。Secret 只从提示或标准输入进入。

前台跑桥，电脑开着、这个进程开着就能答：

```bash
npm i -g lark-channel-bridge
lark-channel-bridge run --app-id <APP_ID> --profile <名字> --agent <claude|codex>
```

已安装的桥如果在 `--help` 里还列出别的 agent 种类，用那个种类。项目说明：https://github.com/zarazhangrui/lark-coding-agent-bridge

只把消息流打进当前对话、由当前 agent 回复时：

```bash
lark-cli event consume im.message.receive_v1 --as bot
```

停监听用 `lark-cli event stop --app-id <APP_ID> --force`。不要先 `kill -9`。不要结束用户的飞书客户端。不要为了查看在线连接数再开一条 `event consume`。

隔一段时间拉取聊天记录不是即时回答。

## 24 小时路径

只在用户确认条件之后做。换密码和两份 Secret 见 [reference.md](reference.md)。

1. 在那台一直开着的机器上安装 Node.js 20.12 或更高、`lark-cli`、Agent，并完成 Agent 自己的登录。`npm i -g lark-channel-bridge`。服务命令不要用 `npx`。
2. 用已有应用初始化，不要让首次向导再创建一个应用：`lark-channel-bridge run --app-id <APP_ID> --profile <名字> --agent <claude|codex>`。
3. 前台日志出现已连接后停掉前台，改成系统服务：`lark-channel-bridge start --profile <名字>`。Linux 上先执行 `sudo loginctl enable-linger <登录用户>`。
4. 停掉其他机器上这个 App ID 的监听：`lark-cli event stop --app-id <APP_ID> --force`。
5. 再按上一节确认机器人在群里，并且已经发过 `/invite group`。
6. 请用户 @ 机器人。处理卡片会先出现，完整回复要等 Agent 跑完。日志里 `queue=` 接近 `0ms`，表示消息已经即时进桥。

群里的 Agent 还要自己调飞书接口时，按 reference 里的「两份密码」再写一份。`secrets set` 只更新监听用的那一份。

## 完成标准

- 目标群成员里能看到这个机器人。
- 所有者之外的人要提问时，群里已经发过 `/invite group`。
- 有人 @ 之后有回复。
- 用户确认了 24 小时时：常驻服务是 active，本次启动之后的日志有「已连接」和「正在监听消息」，没有第二台机器占着同一个 App ID。

## 不要做

- 不要把 App Secret 写进仓库、命令参数、日志或群聊，也不要在回复里复述。
- 不要在用户确认之前安装 24 小时服务。
- 不要为了同一条监听线新建第二个应用。
- 不要在个人电脑和常驻机器同时听同一个应用。
- 不要假设群里的 Agent 看得到操作者电脑上的文件或 `127.0.0.1`。文件放到它的工作目录，或发到群里。
