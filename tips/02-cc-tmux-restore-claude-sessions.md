# 服务器一重启，tmux 里开的 Claude Code 全没了？用 cc-tmux 一条命令全部恢复

2026-11-12 · Jianxiang He · 工具仓库：[Jacksonha7/cc-tmux](https://github.com/Jacksonha7/cc-tmux)

## 这个工具解决什么问题

在学校的集群或者公司的服务器上做实验，我习惯在 tmux 里开好几个窗口，每个窗口里跑一个 Claude Code：一个在改论文，一个在调训练脚本，一个在看日志。问题是共享的登录节点会不定期重启，一重启 tmux 就没了，所有窗口一起消失，哪个对话原来在哪个窗口、哪个目录，也都跟着没了。

其实 Claude Code 会把每段对话存在硬盘上，用 `claude --resume <会话 id>` 就能接着聊，但这个 id 得自己一个个去翻，还要记得每个对话原来在哪个目录。cc-tmux 就是帮你记这本账的：平时自动记下每个 tmux 窗格里跑的是哪段 Claude 对话，重启之后一条命令把 tmux 的会话和窗口重新建好，并在每个窗格里自动接上原来的对话。

```text
重启前                                   重启后，运行 cc-tmux restore
tmux 会话 main                           tmux 会话 main（同名，自动重建）
├── 窗口 paper   → claude（改论文）        ├── 窗口 paper   → claude --resume <原会话>
├── 窗口 train   → claude（调脚本）        ├── 窗口 train   → claude --resume <原会话>
│                → claude（看日志）        │                → claude --resume <原会话>
└── 窗口 notes   → 普通 shell             └── （没跑 claude 的窗格不恢复）
```

需要说清楚的一点：它恢复的是跑着 Claude Code 的那些窗格，没有运行 claude 的普通窗格不会被恢复。

## 先搞懂它是怎么工作的

整个工具就三个命令：

```text
cc-tmux save      扫描所有窗格，记下每个 claude 的会话 id（由 tmux 状态栏定时自动调用）
cc-tmux list      查看上一次开机时留下的快照
cc-tmux restore   按快照重建 tmux 会话和窗口，每个窗格里自动 claude --resume
```

平时它是这样记账的：

1. 安装时会在 tmux 的状态栏里挂一个钩子，状态栏每 15 秒刷新一次，每次刷新都会悄悄运行一次 `cc-tmux save`。它不输出任何东西，所以状态栏看起来没有任何变化。
2. `save` 会逐个检查 tmux 的窗格，顺着窗格的进程树往下找（最多找三层），看里面有没有在跑 claude。
3. 找到之后，去读 Claude Code 自己为每个运行中的会话写的记录文件 `~/.claude/sessions/<进程号>.json`，拿到会话 id、工作目录和会话名。
4. 把这些信息写进一个快照文件。每台机器每次开机各对应一个文件，文件名里带着主机名和这次开机的 id。

```text
~/.cache/cc-tmux/
├── login01-<上次开机的 id>.json   ← 重启前最后一次保存的快照，restore 用的就是它
└── login01-<这次开机的 id>.json   ← 这次开机后新记的
```

重启之后运行 `restore`，它会自动挑出这台机器"上一次开机"留下的最新快照，用原来的名字和目录重建 tmux 会话和窗口。一个窗口里原来有好几个窗格的，会重新切分出来，然后在每个窗格里执行 `claude --resume <会话 id>`。

还有两个保护措施：一是如果某个时刻一个 claude 都没在跑，就不会覆盖已有的快照；二是会检查进程的启动时间，避免进程号被复用时读到别人的旧记录。

## 准备工作

| 要准备的 | 说明 |
| --- | --- |
| Linux 服务器 | 工具依赖 Linux 的 `/proc`，macOS 本机用不了 |
| Python 3.8 及以上 | 服务器上一般自带 |
| tmux 3.x | 运行 `tmux -V` 查看版本 |
| Claude Code 2.1 及以上 | 运行 `claude --version` 查看；它会为每个运行中的会话写 `~/.claude/sessions/<进程号>.json`，cc-tmux 读的就是这个 |

## 第一步：安装

在服务器上运行：

```bash
git clone https://github.com/Jacksonha7/cc-tmux.git
cd cc-tmux
./install.sh
tmux source-file ~/.tmux.conf      # 让 tmux 重新读一遍配置，或者直接重启 tmux
```

`install.sh` 只做两件事：一是把脚本复制到 `~/.local/bin/cc-tmux`，二是在 `~/.tmux.conf` 末尾加上状态栏钩子。如果钩子已经加过，就不会重复添加。它加进去的是这三行：

```bash
# cc-tmux: snapshot the Claude Code session in every pane every 15 s (prints nothing)
set -g status-interval 15
set -ag status-right '#(python3 ~/.local/bin/cc-tmux save)'
```

如果安装时提示 `~/.local/bin` 不在 PATH 里，按提示把下面这行加进 `~/.bashrc` 或 `~/.zshrc`，再重新打开终端：

```bash
export PATH="$HOME/.local/bin:$PATH"
```

## 第二步：平时什么都不用做

装好之后照常在 tmux 里开窗口、跑 Claude Code 就行，状态栏会每 15 秒自动保存一次快照。想确认它有没有在工作，可以看一眼快照目录：

```bash
ls ~/.cache/cc-tmux/
```

能看到一个以主机名开头的 `.json` 文件，就说明在正常记账。

## 第三步：服务器重启之后

重新登录服务器，先看看重启前都跑了些什么：

```bash
cc-tmux list
```

输出每行是一个窗格：`会话:窗口.窗格  会话名  工作目录  会话 id`。确认没问题后一键恢复：

```bash
cc-tmux restore
tmux attach
```

`restore` 会逐行打印恢复了哪些对话，最后告诉你一共恢复了几个。`tmux attach` 进去之后，每个窗格停在对应对话的恢复界面，按回车就能接着聊。

如果想恢复更早的某一次，可以把快照文件路径直接传进去：

```bash
cc-tmux restore ~/.cache/cc-tmux/<某个快照>.json
```

快照都是很小的 JSON 文件，每次开机一个，旧的随时可以删。

## 目前的限制

| 限制 | 说明 |
| --- | --- |
| 只恢复跑着 claude 的窗格 | 没有运行 claude 的普通窗格不会被重建 |
| 布局恢复成平铺 | 一个窗口里有多个窗格时，恢复成 tmux 的 `tiled` 布局，不是原来的精确尺寸 |
| 进程树只找三层 | claude 必须是窗格里 shell 的三层以内的子进程，嵌套更深的找不到 |
| 依赖 Claude Code 的内部文件格式 | 读的是 Claude Code 自己写的会话记录，格式以后可能变；目前在 tmux 3.4 和 Claude Code 2.1.x 上测过 |

## 参考

- [cc-tmux 仓库（代码和英文说明）](https://github.com/Jacksonha7/cc-tmux)
