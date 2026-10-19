# 飞书群 Agent 参考

给执行 [SKILL.md](SKILL.md) 的 agent。这里只写会做错的地方。

## 加群

调用者必须已经在目标群里。用用户身份，不要用机器人身份去拉自己。

```bash
lark-cli im chat.members create --as user \
  --params '{"chat_id":"<CHAT_ID>","member_id_type":"app_id","succeed_type":1}' \
  --data '{"id_list":["<APP_ID>"]}'
```

Windows 的 PowerShell 会吃掉 JSON 引号。把参数放进 Node 的 `spawn` argv 数组，不要拼进一条带引号的 PowerShell 字符串。

| 现象 | 含义 | 处理 |
| --- | --- | --- |
| `232033` | 外部群还没发布「对外共享」 | 在开放平台发布对外共享，再拉机器人 |
| `232028` | 把外部联系人加进了内部群 | 使用飞书另开的外部群，把机器人加到那个新群 |
| 拉进外部联系人后，原来的机器人不在了 | 飞书另开了一个外部群，机器人不会自动跟过去 | 搜索新群名，把机器人加进新群。旧的内部群还在 |

外部群里，机器人不能当群主。机器人也不能因为某人在群里，就去读对方租户里的私人文档。

群名以用户更正后的字为准，搜不到时用更短的关键词再搜。

## 谁的 @ 会得到回答

桥默认只回答应用所有者。其他人 @ 了，机器人保持沉默，或提示管理员执行 `/invite group`。

所有者在目标群发送：

```text
/invite group
```

这是允许这一个群里的所有成员。允许某个人私聊用 `/invite user @对方`。所有者自己永远可以私聊机器人并发送 `/config`。

改完下一条消息就生效，不用重启。

## 两份密码

重置 App Secret 会立刻挂断这个应用的所有长连接，包括它已经在回答的其他群。先说明影响，得到明确同意再重置。

新 Secret 只从标准输入写入。不要出现在命令参数、文档、群消息或日志里。

监听用的一份在桥的密钥库。非交互终端执行下面的命令，它会读一行密码：

```bash
lark-channel-bridge secrets set --app-id <APP_ID> --profile <名字> >/dev/null
```

有的安装没有这个子命令，程序入口是仓库里的 `dist/cli.js`。先看 `lark-channel-bridge secrets --help`。只核对 id、不看明文：`secrets list --profile <名字>` 里应有 `app-<APP_ID>`。

群里的 Agent 调用 `lark-cli` 时用的是另一份，路径是：

```text
~/.lark-channel/profiles/<名字>/lark-cli/lark-channel/config.json
```

`secrets set` 不会改这一份。所以会出现：群里还能回复，但 Agent 说 `app_secret` 无效。重置后在同一台机器上再执行一次，配置目录指向这个文件夹：

```bash
LARKSUITE_CLI_CONFIG_DIR=~/.lark-channel/profiles/<名字>/lark-cli/lark-channel \
  lark-cli config init --force-init --app-id <APP_ID> --app-secret-stdin --brand feishu
```

不要加 `--new`。写完后执行：

```bash
lark-cli api GET /open-apis/bot/v3/info --as bot
```

`ok` 为 true，接口这份才算换好。国际版 Lark 把 `--brand feishu` 换成对应品牌。

个人电脑上的 lark-cli 把 Secret 放在系统密钥链里。不要把密钥链里的二进制拷到常驻机器。常驻机器只通过 `secrets set` 和上面的 `config init` 写入。

## 换密码后的恢复顺序

1. 把新 Secret 写进常驻机器的两份位置。
2. 在占着线的那台电脑执行 `lark-cli event stop --app-id <APP_ID> --force`。
3. `lark-channel-bridge restart --profile <名字>`。
4. 看本次重启之后的日志，不要看重启前留下的旧错误。

日志在 `~/.lark-channel/profiles/<名字>/logs/daemon/`。标准输出在新进程启动后应有「已连接」和「正在监听消息」。标准错误里成片的 `failed to obtain token`、`could not resolve bot identity`，如果时间早于这次重启，是旧密码留下的。重启之后还在新增，就是新密码没写进去，或另一台机器仍占着线。

删掉「接收消息」并发布，不会踢掉已经连着的长连接。只有重置 App Secret 会把旧连接清零。

飞书只返回在线连接数量，不返回机器名。`event status` 和 `event stop` 只管执行命令的那台电脑。

## 服务没有在听

Linux 单元名是 `lark-channel-bridge.bot.<名字>.service`，由 `lark-channel-bridge start` 安装。

```bash
systemctl --user is-active lark-channel-bridge.bot.<名字>.service
systemctl --user status lark-channel-bridge.bot.<名字>.service
```

SSH 会话里能用 `systemctl --user`，还要 `loginctl enable-linger`。没开 linger 时，人一退出，服务就停。

同一台机器上如果 active profile 是别的名字，操作时始终带 `--profile`，不要依赖「当前选中的那个」。

一个 Agent 的桥不要开到另一台访问不了它模型服务的机器上。连不上模型时，长连接仍然是通的，回复会一直超时。

## 回复慢

消息进桥是即时的，`queue=` 应接近 `0ms`。慢在后面的 Agent 回合。用户先看到的处理卡片不是最终回复。

先确认常驻机器能访问该 Agent 的模型地址。出网超时要换机器或修出网，不要靠多开一条长连接。

群里的 Agent 读不到操作者电脑上的对话，也读不到只在那台电脑上的 `http://127.0.0.1`。
