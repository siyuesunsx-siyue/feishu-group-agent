# feishu-group-agent

让 coding agent 进入飞书群，成员 @ 它就能得到回答。人关掉自己的电脑后仍然要答，监听放在一台一直开着的机器上。

这是给 agent 加载的 skill，不是一个要单独运行的服务。

## 安装

```bash
npx skills add <OWNER>/feishu-group-agent
```

仓库地址确定后，把 `<OWNER>` 换成 GitHub 用户名或组织名。

装好之后，对 agent 说：把某个 agent 加进飞书群，并让群成员可以 @ 它提问。

## 里面有什么

- `SKILL.md`：agent 要执行的步骤
- `reference.md`：外部群、两份 App Secret、服务没在听、回复慢

不要把 App Secret 写进这个仓库。
