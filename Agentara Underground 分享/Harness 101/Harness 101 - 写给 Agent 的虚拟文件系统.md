# Harness 101：写给 Agent 的虚拟文件系统

> 来源: 飞书云文档 (TP1edurd4oKu5DxP73Vc4Pb2nTa)

---

## 前言

如今主流的 Harness / General Agent，底座几乎都是 Coding Agent。Cursor、Claude Code、Devin 抽掉「会写代码」这个能力，剩下的躯壳就薄得只剩一个对话框。底下挂着的，是清一色的 Sandbox——一个能跑 bash、能 npm install、能起一个浏览器的全功能盒子。

但笔者最近在做几个并不需要执行代码的 General Agent 时，开始怀疑这套默认配置。一个写作助手，它需要的是读 Markdown、写 Markdown，外加翻一翻历史草稿；一个日程助手，它要的只是读日历、写日历；一个客服 Agent，干的不过是查工单库、回写状态。这些场景里，Sandbox 是个昂贵的副作用——启动慢、资源占用大、攻击面广，而 Agent 真正需要的，只是 I/O。

于是问题就摆在桌面上：Harness 一定要有基于 Container 或 MicroVM 技术的 Sandbox 吗？能不能把 Sandbox 拆成两层？让需要执行代码的 Agent 用厚的那层，让只需要读写数据的 Agent 用薄的那层？

本文提议一个面向 General Agent 的轻量底座：AFS（Agent File System）。它不替代 Sandbox，而是从 Sandbox 里抽出 I/O 这层，让不需要执行能力的 Agent 不必再为一个完整的盒子买单。

---

## 来自 Linux 的启发

答案其实在 Unix 老哲学里躺了四十年——「一切皆文件」。`/dev/null`、`/proc/self/status`、`/sys/class/net/eth0/address` 这些「看起来是文件，实际是接口」的东西，正是把异构 I/O 抹平成统一抽象的经典招式。

Plan 9 把这套哲学推到极致，连网络连接都是 `/net/tcp/clone` 这样的文件。Agent 时代，我们不妨再来一次：把所有 Agent 需要触碰的世界——用户文档、企业知识库、日历、工单、Session 临时数据——都暴露成虚拟文件，让 Agent 用最朴素的 `read / write / ls / grep` 就能完成绝大多数工作。

---

## 背景

### Sandbox 的成本三件套

把 Sandbox 当作 Agent 的默认底座，代价是实打实的。

1. **启动延迟**。哪怕是最轻的容器化 Sandbox，冷启动也要数百毫秒到数秒。对于一次会话只需要读两个文件、写一段总结的写作助手，这点开销不亚于「为了切一片面包先拉来一台冰箱」。

2. **资源占用**。Sandbox 要预留 CPU、内存、磁盘镜像、网络命名空间。一个空闲实例就要吃掉百兆级内存。

3. **攻击面**。一旦给 Agent 一个能跑任意命令的盒子，安全边界就从「数据访问控制」滑向「容器逃逸防御」。

### 不一定需要 Sandbox 的 Agent，比想象中多

任何一个工作模式可以归纳为「读资源、改资源、写回去」的业务 Agent——销售、HR、法务、运营——都吃不消 Sandbox 的全套设备。它们要的不是「执行能力」，而是「对世界的统一可寻址 I/O」。

### Unix 给过一次答案

「一切皆文件」是 Unix 留给后人的最大遗产之一。stdin / stdout / stderr 是文件，socket 是文件，块设备是文件，/proc 是文件，/sys 是文件。

经典的「伪文件」：
```
/dev/random              # 读它就拿一段熵
/dev/null                # 写它丢掉；读它得 EOF
/proc/cpuinfo            # 读它拿当前 CPU 的型号、核心数、flags
/proc/self/maps          # 读它拿当前进程的虚拟内存映射
/sys/class/net/eth0/address  # 读它拿网卡 MAC 地址
/sys/class/power_supply/BAT0/capacity  # 读它拿电池剩余百分比
```

---

## 设计

### 路径命名空间

AFS 内置三大 namespace，全部以 `/mnt/` 开头：

- `/mnt/user/`：以 user_id 隔离的长期空间。子树下规划好三个保留目录——`/mnt/user/skills/`、`/mnt/user/memory/`、`/mnt/user/workspaces/`
- `/mnt/public/`：无需 user_id 也无需 session_id 的全局只读空间
- `/mnt/session/`：session 级的临时空间，会话结束即销毁

### 核心抽象 AgentFSService

骨架的中枢是 AgentFSService。它对外提供一个 `mount(absolute_path_pattern, provider)` 接口：

```python
mount("/mnt/user/skills/", LocalSkillProvider(...))
mount("/mnt/user/workspaces/drive/", DriveProvider(...))
mount("/mnt/user/workspaces/mail/", MailProvider(...))
mount("/mnt/user/workspaces/tasks/", TaskQueueProvider(...))
mount("/mnt/session/", SessionTmpProvider(...))
```

### Provider 协议：能力靠类型表达

AFS 的做法是把能力做成 Python Protocol：

- **必选**：`read`、`ls`——任何 Provider 都得能读、能列
- **可选**：`write`、`edit`、`grep`、`glob`——通过独立 Protocol 声明

---

## 协议

### Agent 端：AFSClient

```python
class AFSClient:
    def __init__(self, session_id: str) -> None: ...
    def read(self, path: str, *, range: Optional[tuple[int, int]] = None) -> str: ...
    def write(self, path: str, content: str) -> None: ...
    def ls(self, path: str) -> list[str]: ...
    def edit(self, path: str, old: str, new: str) -> None: ...
    def grep(self, path: str, pattern: str) -> Iterator[str]: ...
    def glob(self, path: str, pattern: str) -> list[str]: ...
```

### Provider 端：用 Protocol 声明能力

```python
@runtime_checkable
class BaseProvider(Protocol):
    def read(self, rel_path: str) -> str: ...
    def ls(self, rel_path: str) -> list[str]: ...

@runtime_checkable
class WritableProvider(BaseProvider, Protocol):
    def write(self, rel_path: str, content: str) -> None: ...
    def edit(self, rel_path: str, old: str, new: str) -> None: ...

@runtime_checkable
class SearchableProvider(BaseProvider, Protocol):
    def grep(self, rel_path: str, pattern: str) -> Iterator[str]: ...
    def glob(self, rel_path: str, pattern: str) -> list[str]: ...
```

### 错误语义

| Python 异常 | tool result error code |
|-------------|----------------------|
| FileNotFoundError | not_found |
| PermissionError | forbidden |
| IsADirectoryError | is_a_directory |
| NotImplementedError | unsupported_operation |
| 其它未捕获 | internal_error |

---

## 进阶组合

### 三档怎么选

1. **只读资源 / 简单读写** → 纯 AFS，不挂任何 sandbox
2. **需要跑代码、且语言固定** → 轻沙箱 + AFS
3. **需要 bash、需要任意工具链** → Container Sandbox + AFS

---

## 结语

三条主线：

1. **Sandbox 不是 Agent 的最小公分母，VFS 才是。**
2. **Linux「一切皆文件」的哲学在 Agent 时代再次被验证。**
3. **AFS 真正的价值在 Provider 生态。**

Agent 时代我们最该重学的，不是 Transformer，而是 Unix。模型已经够聪明了；缺的是一套像样的、窄到每个工程师都愿意去实现的接口规范。AFS 只是这条路上的一小步——把「按路径读写」这件事，重新还给 Agent。


## 📷 文档图片

![[img_TP1edurd4oKu5DxP73Vc4Pb2nTa_0_3832_1642.png]]
