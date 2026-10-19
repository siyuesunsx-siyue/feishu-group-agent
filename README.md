# feishu-group-agent

让 coding agent 进入飞书群，成员 @ 它就能得到回答。默认是这台电脑开着就能答。24 小时在线是可选项：要有一台一直开着的机器，并且用户明确同意之后才做。

这是给 agent 加载的 skill，不是一个要单独运行的服务。

## 安装

```bash
npx skills add siyuesunsx-siyue/feishu-group-agent
```

仓库：https://github.com/siyuesunsx-siyue/feishu-group-agent

装好之后，对 agent 说：把某个 agent 加进飞书群，并让群成员可以 @ 它提问。

## 里面有什么

- `SKILL.md`：agent 要执行的步骤
- `reference.md`：外部群、两份 App Secret、服务没在听、回复慢

不要把 App Secret 写进这个仓库。
