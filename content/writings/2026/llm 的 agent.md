---
title: llm 的 agent
date: 2026-06-02
series: llm
---

我最早的时候写了两个 naive 的 agent。一个是帮我总结 arxiv 论文，然后输出到本地 markdown，并打开 kimi 对话框可以进一步讨论。另一个是让 llm 写 python 代码解决问题，本地执行 until success。现在看来就是两脚本，说是 agent 都太浮夸。

Then you have claude code leaked. Openclaws dominant and evade the internet. Tons of discussions on agent mechanism. 以及码农有了自己的 hermes.，然后又造出了 harness 这个概念，真的是卷。

I wonder in essence how it actually works when consuming millions of tokens to accomplish complex tasks. And I am not comfortable 用一个黑盒工具，把一堆本地信息无脑上传。你用了 claude code 应用的那一刻就被绑定上一条浪费 token 之路。So I follow a tutorial to build a mini agent myself. It turns out that agentic behaviour is quite simple and harness is really debatable and mostly 屎上雕一朵早晚会凋零花。

一个技术决定就是为了后续方便集成在 pipeline 里，我用了 python 而不是其它主流 agent 都用的 typescript。以下是 code snippets。

## terminal ui

tui 的最外层就是个 while loop。只配置几个简单 command 拦截。

核心就是这个`agent.generate_stream`方法

```python
@dataclass
class AgentConfig:
    api_key: str = field(default_factory=lambda: os.environ.get("DEEPSEEK_API_KEY"))
    model: str = "deepseek-v4-pro"
    base_url: str = "https://api.deepseek.com"


config = AgentConfig()
agent = Agent(config)

click.secho(
    "Welcome to Agent TUI\n\nCommands\n/clear  clear current session\n/exit   exit the tui\n\ntype your prompt or command"
)
while True:
    try:
        user_input = click.prompt("", prompt_suffix=">")
    except KeyboardInterrupt:
        continue
    except EOFError:
        break

    user_input = user_input.strip()
    if not user_input:
        continue

    if user_input.startswith("/"):
        if user_input == "/exit":
            break

        elif user_input == "/clear":
            click.clear()
            agent = Agent(config)

        else:
            click.echo(
                f'Unknown command: {user_input}. Available: "/exit", "/clear"',
            )
    else:
        for output in agent.generate_stream(user_input):
            pretty_print(output)

```

## generate

先看个 non-stream 的 generate，极简，来自 subagent

1. 初始化对话记录
2. 最大次数内 loop
3. 获得 response
4. 没有 tool call 就 break
5. 每个 tool call，调用工具获得结果
6. tool call 结果拼接在对话记录中
7. 重复 3-6

```python
sub_messages = [{"role": "user", "content": args["prompt"]}]

# no subagent tool call in subagent, to avoid recursion
tools_wo_subagent = [
    tool for tool in self.tools if tool["function"]["name"] != "subagent"
]

for _ in range(self.max_iterations):
    response = self.client.chat.completions.create(
        model=self.model,
        tools=tools_wo_subagent,
        messages=sub_messages,
        extra_body={"thinking": {"type": "enabled"}},
    )
    msg = response.choices[0].message

    tool_calls = msg.tool_calls
    if tool_calls is None:
        break

    sub_messages.append(
        {
            "role": msg.role,
            "reasoning_content": msg.reasoning_content,
            "content": msg.content,
            "tool_calls": msg.tool_calls,
        }
    )

    for tool in tool_calls:
        try:
            args = json.loads(tool.function.arguments)
            func = getattr(self, TOOLS.get(tool.function.name)["handler"])
            result = func(args)
        except Exception as e:
            result = f"error: {str(e)}"

        sub_messages.append(
            {
                "role": "tool",
                "tool_call_id": tool.id,
                "content": json.dumps(result),
            }
        )

```

## tool call

定义支持的 TOOLS，传入`chat.completions.create` 的参数 tools

```python

TOOLS = {
    "read_file": {
        "schema": {
            "name": "read_file",
            "description": "Read a file from disk",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "File path"},
                },
                "required": ["path"],
            },
        },
        "handler": "_read_file",
    },
    "write_file": {
        "schema": {
            "name": "write_file",
            "description": "Write content to a file on disk",
            "parameters": {
                "type": "object",
                "properties": {
                    "path": {"type": "string", "description": "File path"},
                    "content": {"type": "string", "description": "File content"},
                },
                "required": ["path", "content"],
            },
        },
        "handler": "_write_file",
    },
    ...
}
```

每个 tool 都有对应的实现，在 generate 里调用 handler

```python
class Agent
...
    def _read_file(self, args) -> str:
        path = resolve_path(args["path"])
        content = path.read_text()
        content = self.truncate(content)
        return content

    def _write_file(self, args) -> str:
        path = resolve_path(args["path"])
        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_text(args["content"])
        return "ok"

```

## skill 和 system prompt

读取本地 skill 目录，获得 skill

```python
class SkillLoader:
    def __init__(self, skills_dir):
        self.skills = {}
        for f in sorted(skills_dir.rglob("SKILL.md")):
            text = f.read_text()
            meta, body = self._parse(text)
            name = meta.get("name", f.parent.name)
            self.skills[name] = {"meta": meta, "body": body}

    def _parse(self, text):
        r = [t.strip() for t in text.split("---") if t]
        meta = yaml.safe_load(r[0].strip())
        body = r[1].strip()

        return meta, body

    def get_descriptions(self) -> str:
        # cheap one-liners for system prompt
        lines = []
        for name, skill in self.skills.items():
            desc = skill["meta"].get("description", "")
            lines.append(f"- {name}: {desc}")
        return "\n".join(lines)

    def get_content(self, name: str) -> str:
        # full body for skill load_skill
        skill = self.skills.get(name)
        if not skill:
            return f"Error: Unknown skill '{name}'."
        return f'<skill name="{name}">\n{skill["body"]}\n</skill>'
```

然后拼接 skill 的 description 到 system prompt 里

```python
class SystemPromptBuilder:
    def __init__(
        self,
        skill_loader,
    ):
        self.skill_loader = skill_loader

    def build(self):
        # identity
        prompt = "You'are a coding agent.\n"

        # skill
        skills_desc = self.skill_loader.get_descriptions()
        if skills_desc:
            prompt += f"""\n## Skills\n\n{skills_desc}"""

        # instruction from AGENT.md
        prompt += """\n## Instructions

- Do not ...
"""
        return prompt
```

所以还增加了一个 load_skill tool

```python
TOOLS= {
...
    "load_skill": {
        "schema": {
            "name": "load_skill",
            "description": "get the content of a skill",
            "parameters": {
                "type": "object",
                "properties": {
                    "name": {"type": "string", "description": "name of the skill"},
                },
            },
            "required": [
                "name",
            ],
        },
        "handler": "_load_skill",
    },
...
}

class Agent
...
    def _load_skill(self, args):
        name = args["name"]
        return self.skill_loader.get_content(name)
```

## todo

long range task might drift，所以在 generate 里加了个 todo，自动插入 update progress 请求。

```python
TOOLS = {
...
    "todo": {
        "schema": {
            "name": "todo",
            "description": "Update the todo list. Each item has a task and status. Only one item can be in_progress at a time.",
            "parameters": {
                "type": "object",
                "properties": {
                    "items": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "task": {
                                    "type": "string",
                                    "description": "What to do",
                                },
                                "status": {
                                    "type": "string",
                                    "enum": ["pending", "in_progress", "completed"],
                                },
                            },
                            "required": ["task", "status"],
                        },
                        "description": "Full list of todo items (replaces previous list)",
                    },
                },
                "required": ["items"],
            },
        },
        "handler": "_todo",
    },
}


class Agent
...
    def _todo(self, args) -> str:
        items = args["items"]
        in_progress = [i for i in items if i["status"] == "in_progress"]
        if len(in_progress) > 1:
            return "error: only one item can be in_progress at a time"
        self.todos = items
        self.rounds_since_todo = 0

        lines = [f"[{i['status']}] {i['task']}" for i in self.todos]
        return "\n".join(lines)
```

## permission

针对 tool 的 name 和 args 的检查

```python

PERMISSION_RULES = [
    {"behavior": "deny", "tool": "read_file", "path": "\.env"},
    {"behavior": "deny", "tool": "read_file", "path": "\.git.*"},
    {"behavior": "deny", "tool": "read_file", "path": "agent\.py"},
    {"behavior": "deny", "tool": "write_file", "path": "agent\.py"},
    {"behavior": "deny", "tool": "edit_file", "path": "agent\.py"},
    {"behavior": "deny", "tool": "webfetch", "url": "file://.*"},
    {"behavior": "allow", "tool": "webfetch", "url": ".*"},
    {"behavior": "allow", "tool": "read_file", "path": ".*"},
    {"behavior": "allow", "tool": "list_directory", "path": ".*"},
]

class Permission:
    def __init__(self, mode, work_dir):
        self.mode = mode  # 2 modes: BY_RULE or READ_ONLY
        self.work_dir = resolve_path(work_dir)
        self.always_yes_when_ask = False

    def check(self, tool_name, tool_args):
        # safe path
        if (
            tool_name in ["read_file", "edit_file", "write_file", "list_directory"]
            and "path" in tool_args
        ):
            path = resolve_path(tool_args["path"])
            if not path.is_relative_to(self.work_dir):
                return {
                    "behavior": "deny",
                    "reason": f"path '{path}' scapes work dir:",
                }

        # deny
        for rule in PERMISSION_RULES:
            if (
                rule["behavior"] == "deny"
                and rule["tool"] == tool_name
                and self._match(rule, tool_args)
            ):
                return {"behavior": "deny", "reason": "Denied by rule"}

        # allow
        for rule in PERMISSION_RULES:
            if (
                rule["behavior"] == "allow"
                and rule["tool"] == tool_name
                and self._match(rule, tool_args)
            ):
                return {"behavior": "allow", "reason": "Allowed by rule"}

        # mode-based
        if self.mode == "READ_ONLY" and tool_name in ["write_file", "edit_file"]:
            return {"behavior": "deny", "reason": "read only mode: writes blocked"}

        # fall through
        if self.always_yes_when_ask:
            return {
                "behavior": "allow",
                "reason": "Not match any rule. But set always yes when ask.",
            }

        return {"behavior": "ask", "reason": "Not match any rule."}

    def _match(self, rule, tool_args):
        # use key(ex. "path") 's value in rule to build regex to match key (ex. "path") 's value in tool_args
        key = [k for k in rule.keys() if k != "behavior" and k != "tool"][0]

        rule_value = rule[key]
        rule_pattern = re.compile(rule_value)
        arg_value = tool_args[key]
        if key == "path":
            # resolve tool arg value to relative to work_dir, then match pattern
            arg_value = Path(arg_value).resolve().relative_to(self.work_dir)
            arg_value = str(arg_value)

        return rule_pattern.match(arg_value)

    def is_allowed_via_ask(self, reason, tool_name, tool_args):
        req = f"{reason} Allow {tool_name}? yes(y) or no(n) or always yes when ask(a)"
        allow = click.prompt(req, prompt_suffix="").strip()

        if allow == "a":
            self.always_yes_when_ask = True
        return allow == "y" or allow == "a"
```

## hook

这里只实现了 pre tool 和 post tool

```python

HOOKS = {
    # 工具生命周期
    "PRE_TOOL_USE": [{"MATCHER": "READ_FILE", "HOOKS": [pre_tool_hook]}],
    "POST_TOOL_USE": [],
    # "POST_TOOL_USE_FAILURE": [], # 工具执行失败后
    # "PERMISSION_DENIED": [], # 权限被拒绝时
    ...
}

def pre_tool_hook(args):
    return {"block": False, "data": f"notified {args['tool_name']}"}

class Agent
...

    def run_tool_hook(self, hook_name, tool_name, tool_args, tool_result):
        for i in HOOKS.get(hook_name):
            if i["matcher"] == tool_name:
                for hook_func in i["hooks"]:
                    yield hook_func(
                        {
                            "event": hook_name,
                            "tool_name": tool_name,
                            "tool_args": tool_args,
                            "tool_result": tool_result,
                        }
                    )
```

## generate stream

铺垫完了，看看 stream 版。

差别主要就是 stream 的 chunk 拼接。以及 hook、permission、harness 的部分

```python
class Agent
...
    def generate_stream(self, user_input):
        self.messages.append({"role": "user", "content": user_input})

        # clear todo
        self.todos = []
        self.rounds_since_todo = 0

        iter = 0
        while True:
            response = self.client.chat.completions.create(
                model=self.model,
                tools=self.tools,
                messages=self.messages,
                stream=True,
                extra_body={"thinking": {"type": "enabled"}},
            )
            reasoning_content = ""
            content = ""
            tool_calls = []

            for chunk in response:
                delta = chunk.choices[0].delta

                if delta.role is not None:
                    yield {"type": "role", "data": delta.role}
                if delta.content is not None:
                    content += delta.content
                    yield {"type": "content_stream", "data": delta.content}
                if delta.tool_calls is not None:
                    for tc_chunk in delta.tool_calls:
                        while len(tool_calls) <= tc_chunk.index:
                            index = len(tool_calls)
                            tool_calls.append(
                                {
                                    "id": "",
                                    "function": {"name": "", "arguments": ""},
                                    "type": "function",
                                    "iterindex": index,
                                }
                            )
                        tc = tool_calls[tc_chunk.index]
                        if tc_chunk.id:
                            tc["id"] += tc_chunk.id
                        if tc_chunk.function.name:
                            # name is not streamed
                            tc["function"]["name"] = tc_chunk.function.name
                            yield {"type": "tool_name", "data": tc_chunk.function.name}

                        if tc_chunk.function.arguments:
                            tc["function"]["arguments"] += tc_chunk.function.arguments
                            yield {
                                "type": "tool_arguments_stream",
                                "data": tc_chunk.function.arguments,
                            }
                if delta.tool_calls is None and delta.reasoning_content is not None:
                    reasoning_content += delta.reasoning_content
                    yield {
                        "type": "reasoning_stream",
                        "data": delta.reasoning_content,
                    }

            if len(tool_calls) == 0:
                yield {"type": "info", "data": "done"}
                break

            self.messages.append(
                {
                    "role": "assistant",
                    "reasoning_content": reasoning_content,
                    "content": content,
                    "tool_calls": tool_calls,
                }
            )

            for tool in tool_calls:
                self.tool_calls[tool["id"]] = tool
                try:
                    args = json.loads(tool["function"]["arguments"])
                    func_name = tool["function"]["name"]
                    func = getattr(self, TOOLS.get(func_name)["handler"])

                    skip_tool_run = False

                    # pre_tool_use hook
                    ...


                    # permission
                    ...

                    if not skip_tool_run:
                        result = func(args)

                    # post_tool_use hook, annotate or observe, no block
                    ...

                except Exception as e:
                    result = f"error: {str(e)}"

                yield {"type": "tool_result", "data": json.dumps(result, indent=2)}

                self.messages.append(
                    {
                        "role": "tool",
                        "tool_call_id": tool["id"],
                        "content": json.dumps(result),
                    }

                )
            # harness
            # todo
            self.rounds_since_todo += 1
            if self.rounds_since_todo >= 3:
                m = "Please update progress with the todo tool and continue."
                yield {"type": "role", "data": "user"}
                yield {"type": "prompt", "data": m}
                self.messages.append(
                    {
                        "role": "user",
                        "content": m,
                    }
                )

            iter += 1
            if iter >= self.max_iterations:
                yield {"type": "error", "data": "Max iteration reached."}
                break


```

## no

- 没有 memory，也没有 cross-session memory，因为我觉得太复杂了，我只想造 bot
- 最终实现里没有 subagent，因为懒而没有在 agent 实现 permission，导致有安全问题
- 没有 micor-compact
  - deepseek 里的 cache hit/cache miss = 50 = 0.28/0.0014，所有 append only 更好
  - 虽然我实现过，但它会改写历史对话，就 comment out 掉了

```python
tool_result_kept = 0
for m in reversed(self.messages):
    if m["role"] != "tool":
        continue

    # keep last 3
    if tool_result_kept < 3:
        tool_result_kept += 1
        continue

    # replace with placeholder
    tool_name = self.tool_calls[m["tool_call_id"]]["function"]["name"]
    if not m["content"].startswith("[Previous"):
        yield {
            "type": "info",
            "data": f"{tool_name} result compacted",
        }
        m["content"] = f"[Previous: used {tool_name}]"
```

- 没有用 llm 来总结实现上下文压缩，这个更复杂，还会丢 info

## now what

写完了 agent，它躺在那里几天了。

其实最初我的想法是要用它来写 alpha 公式，调用 backtest 评测，从而可以自动挖掘 alpha。类似 autoresearch。
