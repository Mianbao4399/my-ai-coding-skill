# Python后端开发知识体系（面向 Agent 服务·新手友好版）

> 覆盖：Python 环境与工具链、框架选型（为什么 Agent 服务用 FastAPI，Flask/Django 带过）、FastAPI 实战线、Agent 服务怎么跑起来（循环/工具/流式）、接入 App 聊天页（WebSocket、消息协议与数据接口路径选型）、查看 Agent 思考过程并调整业务流程、部署、报错排查、全景 SOP。
> 阅读方式：每个概念先讲"是什么（带比喻）"，再讲"为什么需要它"，最后讲"怎么用/怎么排查"。专业名词均标注中英文。
> 本文档面向的读者已具备基本编程常识，不再解释"什么是前后端"这类入门概念；重心放在**你真正要写的 Agent 服务**上。配套阅读：仓库内《Agent服务核心设计与实现知识体系》（Agent 细节）、《Flutter聊天窗口与Agent接入知识体系》（App 端）、《业务API工具化对接知识体系》（工具封装）。
> 信息核验日期：2026-11-27。版本号、价格等动态信息以官方页面为准（文末附链接）。

---

## 目录

- 第 0 章：一组比喻建立全局
- 第 一 章：一次请求的一生（服务架构速览）
- 第 二 章：动手前的准备——Python 环境、虚拟环境、pip
- 第 三 章：框架选型——Agent 服务为什么用 FastAPI（Flask/Django 带过）
- 第 四 章：FastAPI 实战线——从零写一个待办 API
- 第 五 章：Agent 服务是怎么跑起来的（循环、工具、流式）
- 第 六 章：把 Agent 接进 App 聊天页（WebSocket、消息协议与数据接口路径选型）
- 第 七 章：查看 Agent 的思考过程——调试业务流程
- 第 八 章：上线部署一页通
- 第 九 章：全景 SOP（从零到跑通 Agent 服务）
- 第 十 章：常见报错与排查
- 第 十一章：高频速查表
- 第 十二章：进阶与业务视角
- 附录：实操入口地图与排障动手顺序

---

## 0. 先用一组比喻建立全局

| 技术概念 | 英文 | 比喻 | 一句话职责 |
|---|---|---|---|
| HTTP | HyperText Transfer Protocol | 点单用的标准话术 | 客户端与服务器通话的协议 |
| API | Application Programming Interface | 服务窗口 | 约定好的"怎么请求、返回什么" |
| JSON | JavaScript Object Notation | 标准打包盒 | 数据在网络上传来传去的通用格式 |
| Web 框架 | Web Framework | 精装修厨房 + 菜谱模板 | 帮你少写重复代码的工具箱（本项目：FastAPI） |
| 路由 | Routing | 包间号→厨师分配表 | 决定一个网址/连接由哪个函数处理 |
| ORM | Object-Relational Mapping | 翻译官 | 在 Python 类和数据库表之间互相翻译 |
| Pydantic | — | 验货员 | 进出数据先验一验，不合格就拦下 |
| WSGI | Web Server Gateway Interface | 传统同步灶台 | 一个请求占一个灶眼、做完再接单的旧式约定 |
| ASGI | Asynchronous Server Gateway Interface | 支持异步的新灶台 | 同时照看很多单子、谁好了喊谁的约定 |
| 虚拟环境 | Virtual Environment | 每家自用的工具箱 | 让每个项目的第三方包互不干扰 |
| pip | — | 采购员 | 去 PyPI 仓库把第三方包买回来装上 |
| Nginx | — | 门卫 + 传菜口 | 生产环境挡在最前面收请求、发静态文件的软件 |
| uvicorn | — | 真正开火的灶台 | 跑起你的 Python 代码、对外提供服务的运行器 |
| LLM API | Large Language Model API | 顾问热线 | 把对话发给大模型，拿回回答 |
| 工具调用 | Tool Calling | 顾问打内线电话查账本 | 模型决定何时调用你注册的函数（查库、调业务 API） |
| 流式输出 | Streaming | 顾问边想边说 | 回答逐段推给 App，用户不用干等 |
| 思维链 | reasoning_content / Chain-of-Thought | 模型的草稿纸 | 模型回答前的推理过程，调试业务流程全靠它 |
| 消息协议 | Message Protocol | 对讲机暗号表 | App 与 Agent 服务互发消息的类型约定 |

一句话把主线串起来：**App 聊天页与你的 FastAPI 服务保持一条"对讲机"长连接，Agent 循环（主厨）拿着菜单（工具清单）问大模型（顾问）该干什么，模型点头就打内线电话（工具调用）查业务数据，每走一步都按暗号表（消息协议）向 App 播报，模型的草稿纸（reasoning_content）和你的日志就是你调业务流程的依据。**

---

## 一、一次请求的一生（服务架构速览）

理解这张图，后面所有章节都是在给图里的某一段做文章：

```
App 聊天页（Flutter）
   │  ① WebSocket 长连接（双向对讲机，详见第六章）
   ▼
FastAPI 服务（/ws/chat 端点：握手、会话、鉴权）
   ▼
uvicorn（灶台：真正运行 Python 代码）
   ▼
Agent 循环（主厨：想 → 做 → 看，详见第五章）      ← 业务流程代码在这里
   ├──► LLM API（DeepSeek：推理 + 决定用哪个工具）
   └──► 业务工具（查故障码、查配件、下单——即你的业务 API，详见《业务API工具化对接》）
   │
   └──② 每跑一步就推一条消息（text_delta / tool_status / card）原路返回 App
```

每层都有"为什么"：

1. **uvicorn 是运行器**：你写的代码自己跑不起来对外服务，需要它监听端口并调用你的函数。
2. **Agent 循环是主厨**：普通的 API 是"一问一答"，Agent 是"想一步、做一步、看结果、再想下一步"的循环——业务流程就是写在循环里的。
3. **LLM API 是顾问**：模型不直接操作你的数据，它只能"提议"调用哪个工具，真正执行权在你手里——这是安全边界，也是业务可控的原因。
4. **消息推送是体验**：Agent 一步要跑几秒，边跑边播报（流式）才不会让用户觉得卡死。

日常迭代循环：产品/自己提需求 → 改 Agent 的提示词、工具或流程代码 → 用第七章的思考轨迹自测 → 和 App 联调 → 部署。

---

## 二、动手前的准备：Python 环境与工具链

### 2.1 Python 版本怎么选

Python 是解释型语言，先装"解释器"（Interpreter，真正运行你代码的程序）。当前版本事实（核验于 2026-11-27，以 [python.org/downloads](https://www.python.org/downloads/) 为准）：

| 项目 | 现状 |
|---|---|
| Python 最新稳定系列 | 3.15（2026-10-09 发布） |
| 推荐安装 | **3.13 或 3.14**：FastAPI、DeepSeek SDK 等主流库适配最稳 |

> **新手常见疑问：装最新的 3.15 最好吗？**
> 不一定。第三方库适配新版本需要时间，版本太新容易踩到某个包"还没支持"。3.13/3.14 生态兼容最稳。

**怎么装**：打开 <https://www.python.org/downloads/> → 下载安装包。Windows 安装界面务必勾选 **"Add python.exe to PATH"**（否则命令行敲 `py` 会找不到命令）。装完验证：

```
py --version          # Windows：查看版本
python3 --version     # macOS / Linux：查看版本
```

### 2.2 虚拟环境 venv：每个项目一个独立工具箱

**是什么**：虚拟环境（Virtual Environment）是把 Python 和第三方包"复制"到项目文件夹里的机制。

**为什么需要**：A 项目要这个版本的包、B 项目要那个版本，全装在系统里必然打架。虚拟环境让每个项目有自己的"工具箱"。

**怎么用**（逐条注释）：

```
# Windows（py 是 Windows 上的 Python 启动器）
py -m venv .venv                # 在当前目录创建名为 .venv 的虚拟环境
.venv\Scripts\activate          # 激活：之后终端里的 python/pip 都指向这个工具箱

# macOS / Linux
python3 -m venv .venv
source .venv/bin/activate
```

激活成功的标志：命令行开头出现 `(.venv)` 前缀。退出用 `deactivate`。

**预防**：新建项目第一件事就是 venv + 激活，再装包。VS Code 里 `Ctrl+Shift+P`（macOS 为 `Cmd+Shift+P`）→ **Python: Select Interpreter** → 选中 `.venv`，编辑器和终端就用同一套工具箱。

### 2.3 pip：采购员

pip 是包管理器（Package Manager），从 [PyPI](https://pypi.org/) 下载安装第三方包。高频命令：

```
pip install "fastapi[standard]" # 安装 FastAPI 及官方标准配件（含 uvicorn）
pip install openai              # DeepSeek 等厂商兼容 OpenAI 协议，用同一个 SDK
pip install Django==6.1.3       # 精确指定版本
pip list                        # 看这个环境里装了什么
```

国内网络慢时，可指定清华镜像源（镜像地址以 [清华镜像官方页](https://pypi.tuna.tsinghua.edu.cn/) 为准）：

```
pip install openai -i https://pypi.tuna.tsinghua.edu.cn/simple   # -i 指定下载仓库
```

### 2.4 编辑器

[VS Code](https://code.visualstudio.com/) + 官方 Python 扩展（扩展面板搜索 Python 安装）即可起步，新手不必纠结。

---

## 三、框架选型：Agent 服务为什么用 FastAPI

### 3.1 FastAPI 是什么

**是什么**：FastAPI 是一个现代 Python Web 框架，专攻"构建 API"。当前版本 0.165.0（核验于 2026-11-27，以 [PyPI](https://pypi.org/project/fastapi/) 为准）。它像一辆**为 API 而生的轻量跑车**——不替你把家装修好，但开起来又快又顺手。

它由两个底座撑起来：

| 组件 | 比喻 | 职责 |
|---|---|---|
| Starlette | 车的底盘 | 处理路由、请求、响应这些网络底层活（含 WebSocket） |
| Pydantic | 验货员 | 按你声明的类型自动校验数据，不合格直接拦下 |
| uvicorn | 灶台 | 运行器，真正把服务跑起来（标准配件自带） |

三个招牌能力：

1. **类型注解即校验**（Type Hints，Python 自带的参数类型标记）。函数签名里写 `item_id: int`，框架自动把网址里的字符串转成整数，转不动就返回清晰错误。
2. **自动交互文档**：启动后访问 `/docs` 得到 Swagger UI（可点按钮直接试接口），`/redoc` 是另一份排版文档。底层是 OpenAPI 标准，文档不用手写。
3. **异步与 WebSocket 一等公民**：基于 ASGI 新灶台，接口可写成 `async def`，原生支持 WebSocket 路由——聊天服务要的长连接、流式推送正是它的主场。

### 3.2 Flask 与 Django：一眼带过

**Flask**（当前 3.1.3，核验于 2026-11-27，[官方文档](https://flask.palletsprojects.com/en/stable/)）：轻量级"微框架"，像一辆**折叠自行车**——出厂只有三个零件（Werkzeug 处理请求、Jinja 模板、Click 命令行），数据库、登录、迁移全靠扩展（Extension，社区插件包）按需自装。9 行代码能起服务，默认端口 **5000**。适合小工具、微服务这类"要最小自由度"的场景。

**Django**（当前 6.1.3，5.2 LTS 服务到 2028 年 4 月，[官方下载页](https://www.djangoproject.com/download/)）："电池全配齐"的全功能框架，像一间**拎包入住的全装修公寓**——ORM、Admin 管理后台、用户认证、表单、模板出厂自带，MVT（Model-Template-View）结构，适合"有页面、有后台、有权限"的企业网站；企业存量项目多，老项目维护时会遇到它。做 API 要加 Django REST Framework（DRF）扩展。

两者都能胜任一般后端，但都不是为 Agent 服务而生——本项目选 FastAPI，理由见 3.4。

### 3.3 其他 Python Web 框架一眼表

| 框架 | 一句话定位 | 什么时候会见到 |
|---|---|---|
| Tornado | 老牌异步框架，自带服务器 | 高并发长连接的老项目 |
| Sanic | 全 async/await 的高性能框架 | 追求吞吐量的 API 服务 |
| aiohttp | 异步 HTTP 客户端 + 服务端二合一 | 爬虫、代理转发类中间层 |
| Litestar | 与 FastAPI 同思路的新一代框架 | 新项目里偶尔见到 |
| Bottle | 单文件极简框架 | 教学、超小脚本 |

认得名字即可，主攻 FastAPI，认知已覆盖本项目全部场景。

### 3.4 结论：Agent 服务选 FastAPI 的四个理由

1. **WebSocket 与异步一等公民**：聊天服务要长连接、要边跑边推（第五、六章），FastAPI 原生支持，Django 这块生态偏弱、Flask 需要额外配件。
2. **Pydantic 就是工具参数的验货员**：Agent 的每个工具都有一套参数模型（5.3），Pydantic 声明即校验，模型填错参数直接拦下。
3. **自动文档加速调试**：HTTP 调试接口在 `/docs` 点一下就能试，不用另写测试页。
4. **AI 生态同门**：Pydantic AI、LangGraph、各厂商 SDK 在 FastAPI/Pydantic 体系里配合最顺（见 3.5）。

按项目需求速查（收藏这张）：

| 项目需求 | 推荐 | 理由 |
|---|---|---|
| **Agent / LLM 应用后端（本项目）** | **FastAPI** | 异步 + WebSocket + 工具参数校验 + AI 生态 |
| App / 小程序 / AI 服务的纯 API 后端 | FastAPI | 类型校验 + 自动文档 + 异步 |
| 带页面、后台、权限的完整网站 | Django | Admin、ORM、认证全自带 |
| 小工具、微服务、几百行的轻量服务 | Flask | 零约束，按需装件 |
| 第一次学后端、跑通最小 API | FastAPI | 半天跑通，自带文档（见第四章） |
| 极高并发 IO、网关推送类 | Sanic / Tornado | 进阶方向，本项目用不上 |

### 3.5 Agent 开发用 Python 还是 TypeScript

两大生态现状（核验于 2026-11-27，来源见文末）：

| 生态 | 代表框架 | 特点 |
|---|---|---|
| Python | LangChain（模型/工具抽象）、LangGraph（编排运行时：持久化、流式、人机协同）、Pydantic AI、CrewAI / AutoGen（多智能体）、OpenAI Agents SDK | **生态最深**：教程、踩坑答案最多，AI 新东西几乎 Python 先到 |
| TypeScript | Mastra（TS 侧领先全家桶）、Vercel AI SDK（Next.js/Node 加 AI 能力首选）、LangGraph.js、OpenAI Agents SDK（双语言版） | **与前端同栈**：流式 UI、Node/Edge 部署；与 Python 的差距在快速缩小 |

三问决策：① 更熟哪门语言？熟 TS 全栈就直接 TS。② Agent 是独立服务还是嵌在 Next.js 网页里？独立服务 → Python 顺手。③ 需要最新最全的 AI 生态（RAG、评测、多智能体）？→ Python 优先。

**默认答案：学 Agent 后端先学 Python**，概念两边互通、TS 随时可补。本项目已选 Python + FastAPI：Agent 服务是独立 Python 微服务，对上 WebSocket 服务 App，对下工具封装调用业务 API；且 Agent 服务是 I/O 密集型（时间都花在等模型和等业务 API），语言速度差不构成选型依据——《Agent服务核心设计与实现知识体系》里有完整论证。

---

## 四、FastAPI 实战线：从零写一个待办 API

> 本章是执行线，概念在第三章。这个"待办 API"练的是 FastAPI 手感——路由、参数校验、启动，和后面 Agent 服务用的是同一套骨架。

### 4.0 全景步骤总览（收藏这张）

```
第 0 步 [装好 Python + 编辑器] → 第 1 步 [建项目文件夹 + venv]
→ 第 2 步 [pip 安装 fastapi] → 第 3 步 [写 main.py]
→ 第 4 步 [fastapi dev 启动] → 第 5 步 [浏览器 /docs 试接口]
→ 第 6 步 [加完整增删改查] → 第 7 步 [进阶：接数据库]
```

前置条件：Python 3.13/3.14 已安装（见 2.1）；无需账号、无需费用，全部开源免费。

### 4.1 第 1~2 步：建项目、装包

```
mkdir fastapi-todo              # 建项目文件夹
cd fastapi-todo                 # 进入文件夹
py -m venv .venv                # 创建虚拟环境（macOS/Linux 用 python3 -m venv .venv）
.venv\Scripts\activate          # 激活（macOS/Linux 用 source .venv/bin/activate）
pip install "fastapi[standard]" # 安装 FastAPI + 官方标准配件（含 uvicorn、fastapi-cli）
```

### 4.2 第 3 步：写 main.py

新建 `main.py`（这就是一个完整 API）：

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI(title="待办 API")

class TodoIn(BaseModel):        # 请求体的"验货单"：title 必须有，done 不填默认 False
    title: str
    done: bool = False

todos: dict[int, dict] = {}     # 先用内存字典当仓库，4.5 换成真数据库

@app.get("/todos")              # GET：列出全部待办
def list_todos():
    return list(todos.values())

@app.post("/todos", status_code=201)   # POST：新建一条
def create_todo(todo: TodoIn):
    new_id = max(todos, default=0) + 1
    todos[new_id] = {"id": new_id, **todo.model_dump()}
    return todos[new_id]

@app.put("/todos/{todo_id}")    # PUT：按路径参数更新
def update_todo(todo_id: int, todo: TodoIn):
    if todo_id not in todos:
        return {"error": "not found"}
    todos[todo_id] = {"id": todo_id, **todo.model_dump()}
    return todos[todo_id]

@app.delete("/todos/{todo_id}", status_code=204)  # DELETE：删除
def delete_todo(todo_id: int):
    todos.pop(todo_id, None)
```

逐段理解：`@app.get("/todos")` 是路由装饰器——把网址和处理函数绑在一起；`todo: TodoIn` 的类型标注让 Pydantic 自动验货；`todo_id: int` 让 FastAPI 自动把网址里的 `3` 转成整数。

### 4.3 第 4~5 步：启动、看自动文档

```
fastapi dev main.py     # 开发模式启动：自动重载（改代码保存即生效）
```

启动成功后终端显示 `Uvicorn running on http://127.0.0.1:8000`。浏览器打开：

- `http://127.0.0.1:8000/docs` → Swagger UI：点任意接口 → **Try it out** → 填参数 → **Execute**，不写任何前端就能试接口。
- `http://127.0.0.1:8000/redoc` → 另一种排版纯文档。

`fastapi dev` 是官方 CLI 命令；等价写法 `uvicorn main:app --reload`（uvicorn 命令行写法），两种都会在资料中见到。

### 4.4 第 6 步：完整增删改查已经有了

上面代码已覆盖 REST 四个动词：GET 查、POST 增、PUT 改、DELETE 删。可以自己加"按 id 查询"练手：

```python
@app.get("/todos/{todo_id}")
def get_todo(todo_id: int):
    return todos.get(todo_id, {"error": "not found"})
```

### 4.5 第 7 步（进阶）：接数据库

内存字典重启就清零，真项目用 ORM。FastAPI 官方教程配套 SQLModel（模型类同时当 Pydantic 校验模型用），最小示例：

```python
from sqlmodel import Field, Session, SQLModel, create_engine

class Todo(SQLModel, table=True):    # table=True：这个类对应一张表
    id: int | None = Field(default=None, primary_key=True)
    title: str
    done: bool = False

engine = create_engine("sqlite:///database.db")
SQLModel.metadata.create_all(engine)  # 按模型建表（启动时执行一次）

with Session(engine) as session:      # 一条会话里读写
    session.add(Todo(title="买菜"))
    session.commit()
```

Agent 服务的会话历史落库（多轮记忆）用的就是同一套路子。完整工程结构（多文件分层、依赖注入 `Depends`、Alembic 迁移）跟官方 Tutorial 走：<https://fastapi.tiangolo.com/zh/tutorial/>。

### 4.6 FastAPI 高频报错

**现象一**：

```
ModuleNotFoundError: No module named 'fastapi'
```

原理：终端里的 pip 不是这个项目的 pip——十有八九忘了激活虚拟环境，或激活后换了终端窗口。
**怎么办**：① 看命令行有没有 `(.venv)` 前缀；② 没有就重新激活；③ VS Code 右下角确认解释器指向 `.venv`（见 2.2）。
**预防**：每个终端窗口开新会话后先激活 venv 再干活。

**现象二**：请求返回

```
{"detail":[{"type":"missing","loc":["body","title"],"msg":"Field required"}]}
```

原理：HTTP 422 表示"数据验货没过"——请求体缺字段、类型不对。这不是 bug，是 Pydantic 在正常拦截。
**怎么办**：打开 `/docs`，对照接口的 Schema 检查发送的字段名和类型。
**预防**：联调前先在 `/docs` 用 Try it out 跑通一次，再让 App 照着发。

**现象三**：

```
ERROR: [Errno 98] error while attempting to bind on address ('0.0.0.0', 8000): address already in use
```

原理：8000 端口被上一个没关掉的进程占着（Windows 提示常为 `Only one usage of each socket address`）。
**怎么办**：① `Ctrl+C` 停掉旧服务再启动；② 或换端口：`fastapi dev main.py --port 8001`。

---

## 五、Agent 服务是怎么跑起来的

> Agent 的完整设计（三道安全阀、MCP、记忆系统、抽象层）见《Agent服务核心设计与实现知识体系》；本章用最小代码把"循环 + 工具 + 流式"跑通，让你亲手摸到 Agent 的骨架。

### 5.0 全景步骤总览（收藏这张）

```
第 0 步 [注册模型平台、拿 API Key——唯一付费项]
→ 第 1 步 [venv + pip install openai] → 第 2 步 [写最小 Agent 循环]
→ 第 3 步 [注册第一个业务工具] → 第 4 步 [升级为 WebSocket 流式服务（第六章）]
→ 第 5 步 [接 App 聊天页] → 第 6 步 [看思考轨迹调业务流程（第七章）]
```

### 5.1 第 0 步：模型平台的账号与 API Key（唯一的前置付费项）

Agent 与普通后端最大的区别：要多买一位"远程顾问"（LLM API）。本项目用 DeepSeek：

1. 浏览器打开 DeepSeek 开放平台 <https://platform.deepseek.com/> → 注册账号。
2. 充值：按 token（模型计量的文字单位）计费，调试阶段小额即可，价格与优惠以[官方价格页](https://api-docs.deepseek.com/quick_start/pricing)为准。
3. 创建 API Key（密钥，等于你账户的钥匙）→ **立即复制保存**，平台只显示一次。
4. Key 不写进代码和 Git：存环境变量或 `.env` 文件（`.gitignore` 里排除），运行时读取。设置方法：

```
# Windows PowerShell（只对当前终端会话有效）
$env:DEEPSEEK_API_KEY = "sk-你的密钥"
# Windows CMD
set DEEPSEEK_API_KEY=sk-你的密钥
# macOS / Linux
export DEEPSEEK_API_KEY=sk-你的密钥
```

更省心的做法：项目根目录建 `.env` 文件写一行 `DEEPSEEK_API_KEY=sk-你的密钥`，代码开头用 python-dotenv 加载（先 `pip install python-dotenv`）：

```python
from dotenv import load_dotenv
load_dotenv()          # 把 .env 里的键值读进环境变量，之后 os.environ 正常取
```

注意：终端关了临时环境变量就没了，新开终端要重设；`.env` 方案一劳永逸（记得加进 `.gitignore`，见 8.3）。

> **新手常见疑问：Agent 开发一定要用框架（LangChain 等）吗？**
> 不一定。一个 while 循环 + 模型 SDK 就是最小 Agent；框架解决的是多步编排、状态持久化、多智能体协作这些"长大以后"的问题。先用裸循环搞懂原理（本章），再决定要不要上框架。

### 5.2 第 1~2 步：最小 Agent 循环

先装 SDK（DeepSeek 兼容 OpenAI 协议，用 openai 包 + 换 base_url 即可，写法以 [DeepSeek 官方文档](https://api-docs.deepseek.com/) 为准）：

```
py -m venv .venv && .venv\Scripts\activate
pip install openai pydantic
```

新建 `agent.py`——一个能"想、做、看"的完整循环：

```python
import json, os
from openai import OpenAI

client = OpenAI(                                  # 兼容 OpenAI 协议的客户端
    api_key=os.environ["DEEPSEEK_API_KEY"],
    base_url="https://api.deepseek.com",
)

MODEL_NAME = os.environ.get("DEEPSEEK_MODEL", "deepseek-chat")  # 模型名以官方文档为准

messages = [
    {"role": "system", "content": "你是车务助手，只能通过工具查询真实数据。"},
    {"role": "user", "content": "故障码 P0135 是什么意思？"},
]

while True:
    resp = client.chat.completions.create(
        model=MODEL_NAME,
        messages=messages,
        tools=TOOLS_SCHEMA,        # 工具说明书，5.3 定义
    )
    msg = resp.choices[0].message
    if not msg.tool_calls:         # 模型不再调用工具 = 想好了，循环结束
        print(msg.content)
        break
    messages.append(msg)           # 模型的"调用决定"记入历史
    for call in msg.tool_calls:    # 逐个执行工具
        args = json.loads(call.function.arguments)
        result = run_tool(call.function.name, args)      # 5.3 的注册表查表执行
        messages.append({        # 执行结果回喂给模型——"看"这一步
            "role": "tool",
            "tool_call_id": call.id,
            "content": json.dumps(result, ensure_ascii=False),
        })
```

读懂三件事：**while 循环**就是 Agent 的"想→做→看"；`msg.tool_calls` 为空说明模型认为信息够了；工具结果必须回喂，模型才能基于真实数据继续推理。

### 5.3 第 3 步：注册业务工具——函数 + 参数模型 + 描述

每个工具三样东西同源定义（完整对接约定见《业务API工具化对接知识体系》）：

```python
from pydantic import BaseModel

class QueryFaultArgs(BaseModel):          # 参数"验货单"：模型填的格子
    code: str                             # 故障码，如 P0135

def query_fault(code: str):               # 真正干活的函数（内部包业务 API）
    return {"code": code, "meaning": "氧传感器加热器电路故障"}

TOOL_REGISTRY = {"query_fault": (query_fault, QueryFault)}   # 名册：名字 → (函数, 参数模型)

def run_tool(name: str, args: dict):
    fn, schema = TOOL_REGISTRY[name]
    params = schema(**args)               # Pydantic 验货，模型填错参数这里直接报错
    return fn(**params.model_dump())

TOOLS_SCHEMA = [{                         # 喂给模型的"工具说明书"
    "type": "function",
    "function": {
        "name": "query_fault",
        "description": "按故障码查询含义与处理建议。用户提到具体故障码或故障现象时使用。",
        "parameters": QueryFaultArgs.model_json_schema(),
    },
}]
```

把 5.2 和 5.3 拼进同一个 `agent.py`（工具部分放在循环前面），运行 `py agent.py`：终端最终打出模型基于真实工具数据组织的回答——这就是你的第一个 Agent。想同时看到【思考】与【决定调用】过程，加上 7.2 的打印与 7.3 的日志即可。**description 是模型决定"用不用这个工具"的唯一依据**——写得含糊，模型就乱调或不调；调业务流程，很多时候就是改这一行（见 7.4）。

### 5.4 第 4 步起：流式与多轮记忆

- **流式**：给 `create()` 加 `stream=True` 后，回答会变成一段一段的增量片段（delta），每收到一段就往 App 推一条 `text_delta`——"边跑边播报"。协议细节在 6.2，增量解析的工程化封装见 agent 文档的 provider 抽象层。
- **记忆**：`messages` 列表就是短期记忆（Short-term Memory）——把历史消息一起传给模型，它就"记得"上文。真服务里按（用户 ID + 会话 ID）落库，App 重连时恢复；上下文快满时压缩成摘要，进阶做法见 agent 文档的记忆系统章节。

### 5.5 高频坑：reasoning_content 忘了回传

开启思考模式（Thinking Mode，见 7.2）后，**请求带 `tools` 参数时，历史轮次的 `reasoning_content` 必须原样回传**——这是官方文档明确的行为，漏回传会直接报错。**怎么办**：消息历史入库时把 `reasoning_content` 和 `content` 一起存，回传时一起带上（agent 文档 4.3 节的处理方式）。

---

## 六、把 Agent 接进 App 聊天页

> App 端的 Widget 渲染、状态管理见《Flutter聊天窗口与Agent接入知识体系》；本章讲后端这半张图：连接方式、消息协议、服务端骨架。

### 6.1 连接方式：为什么用 WebSocket

聊天服务与普通 API 的本质区别：不是"请求一次回一次"，而是"边跑边推"。两种主流推送通道：

| 方案 | 英文 | 比喻 | 适用 |
|---|---|---|---|
| WebSocket | — | 双向对讲机 | 双向随时说话：用户插话、取消、发卡片回执都要即时传——**App 聊天页选它** |
| SSE | Server-Sent Events | 单向广播喇叭 | 服务端单向推即可的网页场景，实现更简单 |

本项目结论：**App ↔ Agent 服务用 WebSocket**——用户能随时打断重答、点击卡片回执（`confirm_result`），都需要上行通道。

### 6.2 消息协议：七种消息（与 App 端同一张表）

前后端共用一张"暗号表"——协议合同定稿于《车务Agent项目总览与路线图》第三章（M2 里程碑评审冻结），每条消息都是一个带 `type` 的 JSON：

| type | 方向 | 作用 | 要点 |
|---|---|---|---|
| `user_message` | App → 服务 | 用户发言 + 车辆 ID + 会话 ID | 服务端据此绑定会话并开跑 Agent 循环 |
| `text_delta` | 服务 → App | 回答的增量片段 | 高频消息，一个字一段地推 |
| `tool_status` | 服务 → App | 工具调用过程播报 | "正在查询故障记录…"；取消重答也复用它提示 |
| `card` | 服务 → App | 结构化卡片 | 按 `cardType` 分三种：fault 故障记录卡 / knowledge 知识解读卡 / part 配件商品卡，点击可跳转 |
| `confirm` | 服务 → App | 危险操作（下单等写操作）确认请求 | 必须带 `confirmId`，超时要失效 |
| `confirm_result` | App → 服务 | 用户对确认操作的答复 | 服务端据此继续或终止业务流程 |
| `error` | 服务 → App | 出错告知 | 红字 + 重试按钮；重试 = 重发上一条 `user_message` |

**为什么要定死协议**：前后端可以并行开发——前端照协议先画界面、后端照协议先造假数据，最后联调只需要对暗号。协议 v1 已在 M2 评审冻结，改动协议 = 三端同步改：先改总览篇的合同，再分别动三端代码。

### 6.3 服务端骨架：WebSocket 端点

```python
from fastapi import FastAPI, Query, WebSocket

app = FastAPI()

@app.websocket("/ws/chat")
async def chat(ws: WebSocket, token: str = Query("")):   # 鉴权透传：App 连接时带 ?token=xxx
    await ws.accept()                        # 完成握手（对讲机接通）
    while True:
        msg = await ws.receive_json()        # 收 App 的消息
        if msg["type"] == "user_message":
            async for piece in run_agent_stream(msg):   # 5.2 的循环包成流式
                await ws.send_json(piece)   # text_delta / tool_status / card 边跑边推
        elif msg["type"] == "confirm_result":
            await handle_confirm(msg)       # 继续或终止对应的业务流程
```

`run_agent_stream` 的产出节奏：每生成一段文字推 `text_delta`、每次调工具推 `tool_status`、准备好卡片推 `card`——这就是"流式"的全部含义：**不等循环跑完，边跑边播报**。生产版的会话管理、鉴权（连接握手时验 token）、并发控制见 agent 文档第六章。

**没有 App 也能自测**——`pip install websockets` 后用十几行脚本模拟 App 连接（字段以协议合同为准，见总览篇第三章）：

```python
import asyncio, json
import websockets

async def main():
    async with websockets.connect("ws://127.0.0.1:8000/ws/chat?token=test") as ws:
        await ws.send(json.dumps({       # user_message：发言 + 车辆 ID + 会话 ID
            "type": "user_message",
            "content": "P0135 是什么意思？",
            "vehicle_id": "VIN-12345",
            "session_id": "s-001",
        }))
        async for raw in ws:                 # 逐条接收服务端推送
            msg = json.loads(raw)
            print(msg["type"], "->", msg)    # 预期先 tool_status，再连续 text_delta，最后 card/结束

asyncio.run(main())
```

脚本里消息按协议类型依次到达，就说明后端链路通了，再接 App 联调；Windows 终端中文乱码可先设 `$env:PYTHONUTF8="1"`。

### 6.4 App 端渲染：前端的半张表

App 收到消息后的渲染规则由 Flutter 文档定义，后端只需要记住三条对应关系：`text_delta` 高频到达，前端只局部刷新"正在打字"的气泡；`tool_status` 是灰色过程提示，无需交互；`card`/`confirm` 承载点击行为，数据结构必须前后端一致（schema 校验）。

### 6.5 断线重连与心跳（必修，不是优化项）

聊天页三大难点之一是"连接不可靠"：切后台、进电梯、弱网都会断线。生命周期约定（与 Flutter 文档 3.2 对齐）：**App 侧**每 30 秒发一次心跳 ping，连续两个周期收不到任何下行（含 pong）就判定假在线、主动断开；**服务端**收到 ping 回 pong、保证任何下行都算活性信号，另外做好三件事：① 会话历史落库，App 重连后按会话 ID 恢复上下文；② 未推送完的消息缓冲或允许重取；③ 及时清理断开的死连接。App 端配套是退避重连（Backoff Reconnect，1 秒起步翻倍、上限 30 秒）。

### 6.6 数据接口的路径选型：Agent 调还是 App 调？

典型问题：用户在聊天页问了一个汽车故障问题，知识库该由 Agent 服务去调，还是 App 端直接调？**答案按场景分两条线**：

```
聊天里的提问（决策型）
  用户提问 → Agent 循环（决定查什么） → 知识库 API（服务端内网） → 融合成回答（text_delta / card）

聊天外的浏览页（取数型）
  打开"故障码大全"页 → App 直连 HTTP → 知识库 API → 页面独立渲染
```

**为什么聊天场景必须走 Agent**：

| 维度 | App 直调 | Agent 工具化调用 |
|---|---|---|
| 谁决定查什么 | 客户端写死逻辑，应对不了自由提问 | LLM 按语义动态决定：查不查、用什么关键词、查不到换词再查 |
| 上下文融合 | JSON 原样展示，与聊天流割裂 | 结合多轮对话组织回答（会话记忆支撑"上次说的那个故障"） |
| 安全 | 知识库接口暴露给公网客户端（可被反编译、抓包） | 收口服务端：知识库保持内网调用，鉴权/限流/脱敏一处搞定 |
| 协议与体验 | 前端另写一套查询展示 | 复用现有协议：先推 `tool_status` 播报，结果用故障卡 `card` |
| 成本与速度 | 不耗 token、快 | 每轮耗 token、慢零点几秒到几秒 |
| 可维护性 | 查询逻辑散在 App 里，改一次发一次版 | 改工具描述/参数模型即生效，还能靠思维链调试（第七章） |

核心判断一句话：**聊天提问是"决策问题"，不是"取数问题"**——查几次、按什么顺序查，取决于语义和上下文，只有 Agent 循环能做。App 拿到静态 JSON，回答质量、追问、指代消解都不存在。

**App 直调什么时候是对的**：与聊天无关的固定入口浏览页（如"故障码大全"）——纯浏览、不依赖对话上下文，这时绕 Agent 一圈纯属浪费（多花 token、多等几秒）。折中方案：App 不直连知识库，改调 **Agent 服务暴露的普通 HTTP 查询接口**（服务端转发知识库）——安全收口与 WebSocket 一致，又不耗 token，适合"浏览型但想统一鉴权"的场景。

**本项目最常见的混合形态**：聊天中 Agent 给出故障卡（`card`）→ 用户点卡片**跳转到详情页** → 详情页数据用普通 HTTP 接口拉取（浏览型，不经过模型）。卡片跳转和对话内查询是两条路径，各走各的——这也印证"Agent 自己不实现业务规则、查数据全由业务 API 说了算"的架构原则。

> **新手常见疑问：怎么快速判断走哪条路径？**
> 三问：① 这次调用需要理解语言/上下文/做决策吗？需要 → Agent 工具。② 结果要融入对话流吗？要 → Agent 工具。③ 是固定入口的纯浏览、与对话无关吗？是 → App 直调或走服务端 HTTP 接口。

---

## 七、查看 Agent 的思考过程——调试业务流程

> Agent 的业务流程不是 if-else 写死的，而是"提示词 + 工具描述 + 模型推理"共同决定的。所以要调流程，先要能**看见模型在想什么**。

### 7.1 思考过程藏在三个地方

| 层 | 是什么 | 怎么看 |
|---|---|---|
| 模型思维链 | 模型回答前的推理草稿（`reasoning_content`） | API 响应字段，代码打印（7.2） |
| 工具调用轨迹 | 每一轮调了哪个工具、传了什么参数、返回什么 | 你自己的结构化日志（7.3） |
| 服务运行日志 | 请求耗时、token 用量、异常堆栈 | logging / 终端 / 日志文件 |

### 7.2 打印思维链：reasoning_content

DeepSeek 的思考模式（Thinking Mode）会让模型回答前先"打草稿"，草稿就是响应里的 `reasoning_content` 字段（核验于 2026-11-27，[官方思考模式文档](https://api-docs.deepseek.com/zh-cn/guides/thinking_mode/)）。默认开启；简单任务追求首字速度时可显式关闭，参数以官方文档为准。打印它：

```python
msg = resp.choices[0].message          # 5.2 循环里拿到的模型回复
if msg.reasoning_content:              # 官方字段：与 content 同级，仅思考模式有值
    print("【思考】", msg.reasoning_content)
print("【回答】", msg.content)
if msg.tool_calls:
    print("【决定调用】", [(c.function.name, c.function.arguments) for c in msg.tool_calls])
```

读思维链的方法：看模型**怎么理解用户意图**、**为什么选了这个工具**、**基于什么信息得出结论**。业务流程不对劲时，草稿纸会直接告诉你模型在哪一步想歪了。

> **新手常见疑问：思维链能当业务数据用吗？**
> 不能。它是模型的推理草稿，格式和内容都不保证稳定，只用于调试与观测；业务逻辑永远挂在工具调用和最终 `content` 上。

### 7.3 每轮循环的结构化日志

在 5.2 的循环里加几行日志，四个字段决定了你排查问题的速度：

```python
import time, logging

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(message)s")
log = logging.getLogger("agent")

t0 = time.perf_counter()
resp = client.chat.completions.create(model=MODEL_NAME, messages=messages, tools=TOOLS_SCHEMA)
msg = resp.choices[0].message          # 与 5.2 循环里同一行
elapsed = time.perf_counter() - t0
usage = resp.usage                     # token 用量——费用埋点就靠它
log.info("第%d轮 | 思考%s字 | 工具=%s | 耗时%.2fs | tokens=%d+%d",
         round_no,
         len(msg.reasoning_content or ""),
         [c.function.name for c in (msg.tool_calls or [])],
         elapsed, usage.prompt_tokens, usage.completion_tokens)
```

工具执行后再补一条：工具名 + 入参 + 返回结果摘要。四个字段各有用途：**轮次**看是否死循环、**工具与参数**看流程走向、**耗时**定位慢在哪、**token 数**管住成本（会话级汇总、超预算告警，见 agent 文档的费用埋点）。

### 7.4 从思考轨迹定位业务流程问题（收藏这张）

| 现象 | 先看哪里 | 通常调什么 |
|---|---|---|
| 工具没被调用（模型自己编答案） | 思维链里模型怎么理解意图 + 工具 `description` | 改 description：写清"什么时候用、输入是什么" |
| 工具被调但参数错 | 日志里的 `arguments` | 改参数模型（Pydantic）或在提示词里补格式说明 |
| 回答与业务数据不符 | 工具返回的 JSON + 思维链里怎么解读 | 改工具返回内容：只给模型需要的字段，别塞无关大对象 |
| 反复调用同一工具不收敛 | 日志轮次 + 思维链 | 加轮数上限、终止条件；把上一轮结果摘要喂回 |
| 语气/格式飘忽 | `system` 提示词 | 收紧角色定义与输出约定 |
| 该快的不快 | 日志耗时列 | 简单任务关思考模式；工具并行发起；减少轮数 |

排障顺序：先看**日志的工具轨迹**（流程对不对）→ 再读**思维链**（理解对不对）→ 最后改**提示词/工具描述**（一行文案经常顶几十行代码）。

### 7.5 进阶：专门的观测平台

日志写进文件后不好检索时，可上 LLM 观测平台：LangSmith（<https://smith.langchain.com/>，LangChain 官方，trace/评测一体）或 Langfuse（<https://langfuse.com/>，开源可自部署）。它们把每轮调用、工具、token、耗时自动串成可视化的 trace（调用轨迹图）。一期项目日志就够用，规模上来再接。

---

## 八、上线部署一页通

### 8.1 开发服务器 ≠ 生产服务器

`fastapi dev` 是**自家小灶**：单进程、无安全加固，官方明确警告不要用于生产。上生产 = 换"商用灶台"（多进程）+ 前面站一个"门卫"（Nginx）。

### 8.2 FastAPI 生产部署（最小可用）

```
pip install uvicorn
uvicorn main:app --host 0.0.0.0 --port 8000 --workers 4
# main:app = main.py 里的 app 对象；--workers 4 = 开 4 个进程并行接单
```

生产推荐直接用官方 CLI：`fastapi run main.py`。标准三件套：云服务器（或容器）→ Nginx 反向代理（挂域名和 HTTPS 证书）→ 进程守护（systemd 或 Docker）。**本项目注意一条**：聊天服务走 WebSocket，Nginx 必须配置 WebSocket 升级头（`Upgrade`/`Connection`），否则握手直接失败——具体配置见 agent 文档 12.6 节。

### 8.3 数据库与环境变量（通用）

- 开发用 SQLite，上生产换 PostgreSQL（官方推荐；也支持 MySQL 等）。FastAPI/SQLModel 改 `create_engine` 连接串即可。
- API Key、业务 API 地址、模型名全部走环境变量或 `.env`（`.gitignore` 排除），密码密钥绝不进代码和 Git。

---

## 九、全景 SOP（从零到跑通 Agent 服务）

> 本节是执行总纲：前面章节是知识点，这里串成一条按序执行线。

### 9.0 全景步骤总览（收藏这张）

```
第 0 步 [环境准备 + 模型平台账号（唯一付费项）]
→ 第 1 步 [FastAPI 手感线：待办 API]
→ 第 2 步 [Agent 化：循环 + 工具 + 思考日志]
→ 第 3 步 [服务化：WebSocket + 消息协议]
→ 第 4 步 [接 App 聊天页联调]
→ 第 5 步 [用思考轨迹调业务流程] → 第 6 步 [部署]
```

### 9.1 第 0 步：环境准备

| 要做的 | 入口 | 说明 |
|---|---|---|
| 装 Python 3.13/3.14 | <https://www.python.org/downloads/> | Windows 勾选 Add python.exe to PATH（见 2.1） |
| 装 VS Code + Python 扩展 | 扩展面板搜 Python | 见 2.4 |
| （国内）配置 pip 镜像 | 见 2.3 | 可选 |
| **注册模型平台 + 充值 + 建 API Key** | <https://platform.deepseek.com/> | **唯一付费项**：按 token 计费，调试期小额即可，价格以官方页为准；Key 存环境变量（见 5.1） |
| 阅读配套文档 | 仓库 docs/agent/、docs/前端/ | Agent 细节与 App 端协议 |

### 9.2 第 1 步：FastAPI 手感线（对应第四章）

1. `mkdir fastapi-todo` → venv → 激活（4.1 / 2.2）
2. `pip install "fastapi[standard]"`（4.1）
3. 写 `main.py`（4.2）→ `fastapi dev main.py`（4.3）
4. 浏览器开 `127.0.0.1:8000/docs`，Try it out 全部接口（4.3~4.4）
5. 进阶：换 SQLModel + SQLite（4.5）

### 9.3 第 2 步：Agent 化（对应第五章）

1. `pip install openai pydantic`，API Key 进环境变量（5.1）
2. 写最小循环 `agent.py`（5.2）
3. 注册第一个业务工具：函数 + Pydantic 参数模型 + description（5.3）
4. 加思维链打印与每轮日志（7.2 / 7.3）——**从第一天就加，别等出问题**
5. 试跑：问一个会触发工具的问题，观察【思考】→【决定调用】→【回答】三段输出

### 9.4 第 3~4 步：服务化与联调（对应第六章）

1. 把循环包进 `/ws/chat` WebSocket 端点，输出改推 `text_delta` / `tool_status` / `card`（6.3）
2. 与 App 按七种消息协议联调（6.2；App 端照 Flutter 文档）
3. 自测：先用 6.3 的模拟脚本连自己的服务（无需 App 在手），先测文本流，再测卡片与确认框
4. 和 App 约定两类数据路径：对话内查询走 Agent 工具，卡片跳转的详情页走 HTTP（6.6）

### 9.5 第 5~6 步：调业务、上线

- 业务流程不对 → 按 7.4 对照表改 description / 参数模型 / 提示词，用思考轨迹验证。
- 上线 → 过第八章部署清单（云服务器 + Nginx + WebSocket 升级头）。

---

## 十、常见报错与排查

> 共同规律：新手期九成故障落在三类——**环境不对**（包装错地方）、**路径/命令不对**（漏一步）、**忘了执行某个动作**（没回传、没启动）。排障先按这三类归位，再看具体报错。

### 10.1 事故一：装了包，运行还是 ModuleNotFoundError

**现象**：

```
ModuleNotFoundError: No module named 'fastapi'
```

**原理（新手版）**：电脑里有多套 Python 和多个"工具箱"，包装进了 A 工具箱，运行时用的却是 B 工具箱。
**怎么办**：① 确认命令行前缀有 `(.venv)`；② 没有就重新激活（2.2）；③ `pip list` 确认包在这个环境里；④ VS Code 右下角解释器选成 `.venv`。
**预防**：新终端先激活 venv；pip 用 `py -m pip`（Windows）保证和当前 Python 配套。

### 10.2 事故二：端口被占用

**现象**：`address already in use`（Windows 常见 `Only one usage of each socket address`）。

**原理**：一个端口同时只能被一个进程监听，上次的服务没停干净。
**怎么办**：① 找到旧终端 `Ctrl+C`；② 或换端口：`fastapi dev main.py --port 8001`。
**预防**：开服务就守着那个终端；多个服务各用各的端口。

### 10.3 事故三：App/网页联调时 CORS 报错

**现象**：浏览器控制台 `... has been blocked by CORS policy ...`（后端日志里请求其实是通的）。

**原理（新手版）**：浏览器的安全规则——网页 A 的脚本默认不许读网页 B 的接口响应，除非 B 明确放行。注意：WebSocket 不受 CORS 这套规则管，这条只影响 HTTP 接口（如 `/docs`、管理接口）。
**怎么办**：后端加放行配置（FastAPI 示例，放在创建 `app` 之后）：

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],  # 前端开发服务器的地址
    allow_methods=["*"],                      # 放行所有 HTTP 动词
    allow_headers=["*"],
)
```

**预防**：联调前先配；用 curl/Postman 测不会触发（是浏览器在拦）。

### 10.4 事故四：SQLite 文件被锁

**现象**：

```
sqlite3.OperationalError: database is locked
```

**原理（新手版）**：SQLite 同一时刻只允许一个写入者，另一个进程正占用着数据库文件。
**怎么办**：① 关掉其他正在跑服务/脚本的终端；② 只留一个开发服务实例；③ 频繁出现说明该换 PostgreSQL。
**预防**：单人开发 SQLite 够用；多人同时写换 PostgreSQL。

### 10.5 事故五：调模型 API 报 429 或余额不足

**现象**：

```
Error code: 429 - rate limit exceeded
```
或
```
Error code: 402 - Insufficient Balance
```

**原理**：429 是限流——同一时间请求太多，被平台挡回；402 是账户余额不够（或 Key 配额用完）。
**怎么办**：① 429：降低并发，失败后等几秒重试（工程做法是指数退避：每次失败翻倍等待）；② 余额：去平台充值页确认余额与计费。
**预防**：循环里天然是串行调用，429 少见；做费用埋点（7.3 的 token 日志），会话结束汇总。

### 10.6 事故六：thinking 模式报错（漏回传 reasoning_content）

**现象**：多轮工具调用时 API 直接报错（官方文档明确：带 `tools` 的请求，历史轮次的 `reasoning_content` 必须完整回传，漏了会报错）。

**原理**：思考模式的草稿是模型上下文的一部分，断了模型就"忘了自己刚才是怎么想的"。
**怎么办**：消息历史入库时把 `reasoning_content` 与 `content` 一起存，回传时一起带上（见 5.5）。
**预防**：封装一个统一的"消息历史"模块，出入库都带全字段，不手工拼消息列表。

---

## 十一、高频速查表

### 11.1 三框架命令对照（主线 FastAPI）

| 动作 | FastAPI（主线） | Flask | Django |
|---|---|---|---|
| 安装 | `pip install "fastapi[standard]"` | `pip install Flask` | `py -m pip install Django` |
| 建项目 | 手建 `main.py` | 手建 `hello.py` | `django-admin startproject` |
| 启动（开发） | `fastapi dev main.py` | `flask --app hello run` | `py manage.py runserver` |
| 默认端口 | 8000 | 5000 | 8000 |
| 接口文档 | `/docs` 自带 | 无，需扩展 | 需 DRF 扩展 |
| 生产启动 | `fastapi run main.py` | gunicorn | gunicorn |

### 11.2 HTTP 状态码速查

| 码 | 含义 | 什么时候遇到 |
|---|---|---|
| 200 | 成功 | 正常返回 |
| 201 / 204 | 已创建 / 无内容 | POST 新建 / DELETE 成功 |
| 400 | 请求有误 | 参数格式错 |
| 401 / 403 | 未登录 / 无权限 | 鉴权拦下 |
| 404 | 找不到 | 网址错、资源不存在 |
| 422 | 数据校验失败 | Pydantic 验货不过（FastAPI 特征） |
| 429 | 请求太频繁 | 模型 API 限流（10.5） |
| 500 | 服务器内部错误 | 你的代码抛异常，看终端/日志 |

### 11.3 venv / pip 速查

| 目的 | 命令 |
|---|---|
| 创建虚拟环境 | `py -m venv .venv`（macOS：`python3 -m venv .venv`） |
| 激活（Windows） | `.venv\Scripts\activate` |
| 激活（macOS/Linux） | `source .venv/bin/activate` |
| 退出虚拟环境 | `deactivate` |
| 看装了什么 | `pip list` |
| 导出依赖清单 | `pip freeze > requirements.txt` |
| 按清单安装 | `pip install -r requirements.txt` |

### 11.4 Agent 调试速查（收藏）

```
看思考过程三步：
① 打印 reasoning_content——模型怎么理解、为什么选这个工具（7.2）
② 看每轮日志——轮次 / 工具名 / 参数 / 耗时 / token（7.3）
③ 对照 7.4 表——改 description / 参数模型 / 提示词，再跑一轮验证
```

---

## 十二、进阶与业务视角（可跳过）

入门后 1~3 个月会陆续遇到的方向：

- **RESTful 规范**：接口设计约定——名词做网址、动词用 HTTP 方法表达、状态码表达结果。业务 API 的工具化封装（第五章）就建立在它之上。
- **认证**：App 连接用 token 鉴权（WebSocket 握手时验证，见 agent 文档）；管理类 HTTP 接口常用 JWT（JSON Web Token，无状态令牌）。
- **测试**：FastAPI 内置 `TestClient` 函数式断言；Agent 侧重"评估"——同一批问题跑 N 次看工具选择对不对（agent 文档的评估与观测章）。
- **异步任务**：发通知、跑报表丢给 Celery（任务队列），别让请求干等。
- **框架化**：循环写熟后，需要持久化编排、人机协同、多 Agent 协作时再上 LangGraph 等框架（3.5 的生态表）。
- **岗位画像**：AI 应用后端常见组合——FastAPI + PostgreSQL + Redis + Docker + 向量数据库；本文第五~七章正是这条线的主干。

---

## 附录：实操入口地图（去哪里动手）

| 要做的事 | 位置（界面/命令） | 要点 |
|---|---|---|
| 下载 Python | <https://www.python.org/downloads/> | Windows 记得勾 PATH |
| 新建项目 | 终端：`mkdir` + `py -m venv .venv` | 每个项目独立目录 |
| 选解释器 | VS Code：`Ctrl+Shift+P` → Python: Select Interpreter | 指向 `.venv` |
| 模型平台（API Key/充值） | <https://platform.deepseek.com/> | Key 只显示一次，存环境变量 |
| FastAPI 交互文档 | 浏览器：`http://127.0.0.1:8000/docs` | Try it out 直接测接口 |
| 看模型 API 细节 | <https://api-docs.deepseek.com/> | 思考模式、工具调用、模型名 |
| 消息协议合同 | 仓库 docs/车务Agent项目总览与路线图.md 第三章 | 七种消息 + 三种卡片 schema，M2 冻结 |

**排障"动手顺序"速查**（收藏）：

```
① 看终端最后 20 行报错——最后一行是结论，往上找第一处自己文件名的堆栈
② 环境三连：venv 激活了吗 → pip list 有这个包吗 → 跑的是这个解释器吗
③ Agent 三连：API 429/余额了吗 → reasoning_content 回传了吗 → 日志里轮次/工具轨迹正常吗
④ 流程不对：按 7.4 表——先看工具轨迹，再读思维链，最后改提示词
```

---

## 官方与参考链接

**语言与框架**

- Python 下载与版本：<https://www.python.org/downloads/>
- FastAPI 官方文档（中文）：<https://fastapi.tiangolo.com/zh/>
- FastAPI 版本发布记录（PyPI）：<https://pypi.org/project/fastapi/>
- FastAPI CLI（dev/run 命令）：<https://fastapi.tiangolo.com/tutorial/fastapi-cli/>
- Flask 官方文档 / 变更日志：<https://flask.palletsprojects.com/en/stable/> / <https://flask.palletsprojects.com/en/stable/changes/>
- Django 官方下载页：<https://www.djangoproject.com/download/>
- uvicorn 官方文档：<https://uvicorn.dev/>
- 菜鸟教程 FastAPI / Flask（中文实践参考）：<https://www.runoob.com/fastapi/fastapi-tutorial.html> / <https://www.runoob.com/flask/flask-tutorial.html>

**模型 API（本项目用）**

- DeepSeek API 文档：<https://api-docs.deepseek.com/>
- DeepSeek 思考模式与 reasoning_content：<https://api-docs.deepseek.com/zh-cn/guides/thinking_mode/>
- DeepSeek 开放平台（注册/充值/Key）：<https://platform.deepseek.com/>

**Agent 生态（3.5 节结论来源）**

- LangGraph（Python）：<https://langchain-ai.github.io/langgraph/>
- LangGraph.js（TypeScript）：<https://langchain-ai.github.io/langgraphjs/>
- Vercel AI SDK：<https://ai-sdk.dev/>
- Mastra：<https://mastra.ai/>
- OpenAI Agents SDK：<https://github.com/openai/openai-agents-python>（JS 版 <https://github.com/openai/openai-agents-js>）
- Agent 框架生态综述（2026 第三方横评）：<https://the-agent-report.com/2026/05/ultimate-guide-open-source-ai-agent-frameworks/>

**观测平台（7.5 节）**

- LangSmith：<https://smith.langchain.com/>
- Langfuse：<https://langfuse.com/>

**仓库内配套文档**

- 《Agent服务核心设计与实现知识体系》：docs/agent/（循环安全阀、工具系统、记忆、WebSocket 服务化、评估观测）
- 《业务API工具化对接知识体系》：docs/后端/（工具封装约定）
- 《Flutter聊天窗口与Agent接入知识体系》：docs/前端/（消息协议前端视角、卡片渲染、断线重连）
