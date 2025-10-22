
--

---

## معماری واقعی Codex (از document شما):

```
[User Input]
      ↓
[Codex CLI - روی Host]  ←→  [Claude API]
      ↓                           ↑
[Local Sandbox]                   |
(Seatbelt/Landlock/Docker)        |
      ↓                           |
[Execute: shell, patch, etc]      |
      ↓                           |
[Results] ────────────────────────┘
      ↓
[Iterate تا Complete]
```

**نکات کلیدی از document:**
- Codex CLI با **Rust** نوشته شده
- روی **machine developer** اجرا میشه (نه داخل container)
- Sandbox فقط برای **اجرای امن دستورات** است
- ابزارها: `shell` (execvp) و `apply_patch` (diff format)
- Agent خودش با Claude صحبت میکنه، نتیجه میگیره، اجرا میکنه، برمیگردونه

---

## معماری صحیح برای نسخه ما:

```
[Web UI] ─→ [Backend API]
                 ↓
        [Agent Orchestrator] ←─────┐
                 ↓                 |
         ┌───────┴────────┐       |
         ↓                ↓       |
    [Agent 1]        [Agent N]    |
         ↓                ↓       |
    [Sandbox 1]     [Sandbox N]  |
         ↓                ↓       |
    [Execute]       [Execute]     |
         ↓                ↓       |
         └────────┬───────┘       |
                  ↓               |
         [Claude via OpenRouter] ─┘
              (یک نقطه ارتباط)
```

---

## جزئیات دقیق هر کامپوننت:

### 1️⃣ Agent Orchestrator (قلب سیستم)

**مسئولیت:**
- دریافت task از user
- ایجاد N agent instance
- مدیریت conversation با Claude
- ارسال tool calls به sandboxes
- جمع‌آوری و merge نتایج

**نه container، نه agent داخل container - فقط یک process روی host**

```javascript
class AgentOrchestrator {
  constructor(taskId, agentCount, codebasePath) {
    this.taskId = taskId
    this.agents = []
    this.claudeClient = new ClaudeClient(OPENROUTER_API_KEY)
    
    // ایجاد N agent
    for (let i = 0; i < agentCount; i++) {
      this.agents.push(new AgentInstance(i, codebasePath))
    }
  }

  async run() {
    // همه agents موازی شروع میکنن
    const promises = this.agents.map(agent => this.runAgent(agent))
    const results = await Promise.allSettled(promises)
    
    // merge results
    return this.mergeResults(results)
  }

  async runAgent(agent) {
    // این agent loop دقیقاً مثل Codex CLI
    let iteration = 0
    const maxIterations = 50

    // System prompt + task
    const messages = [
      { role: 'system', content: this.getSystemPrompt() },
      { role: 'user', content: agent.task }
    ]

    while (iteration < maxIterations) {
      iteration++

      // ─────────────────────────────
      // مرحله 1: صحبت با Claude
      // ─────────────────────────────
      const response = await this.claudeClient.chat(messages, this.getTools())

      // اضافه کردن response به history
      messages.push({
        role: 'assistant',
        content: response.content,
        tool_calls: response.toolCalls
      })

      // emit reasoning به frontend
      this.emitReasoning(agent.id, response.content)

      // اگر tool call نداره، یا done شده یا باید بپرسیم
      if (response.toolCalls.length === 0) {
        if (this.isTaskComplete(response.content)) {
          break
        }
        continue
      }

      // ─────────────────────────────
      // مرحله 2: اجرای tool calls در sandbox
      // ─────────────────────────────
      const toolResults = []
      for (const toolCall of response.toolCalls) {
        const result = await this.executeToolInSandbox(
          agent.sandbox,
          toolCall.name,
          toolCall.arguments
        )

        toolResults.push({
          tool_call_id: toolCall.id,
          output: JSON.stringify(result)
        })

        // emit به frontend
        this.emitToolCall(agent.id, toolCall, result)
      }

      // ─────────────────────────────
      // مرحله 3: برگشت نتیجه به Claude
      // ─────────────────────────────
      messages.push({
        role: 'tool',
        content: JSON.stringify(toolResults)
      })

      // و loop ادامه پیدا میکنه...
    }

    return {
      agentId: agent.id,
      changes: await agent.sandbox.getChanges(),
      iterations: iteration
    }
  }

  async executeToolInSandbox(sandbox, toolName, args) {
    // این فقط دستور را به sandbox میفرسته و نتیجه میگیره
    switch (toolName) {
      case 'shell':
        return await sandbox.executeShell(args.command, args.workdir)
      
      case 'apply_patch':
        return await sandbox.applyPatch(args.patch)
      
      case 'read_file':
        return await sandbox.readFile(args.path)
      
      // ... other tools
    }
  }

  getTools() {
    return [
      {
        name: "shell",
        description: "Execute shell command in sandbox",
        parameters: {
          type: "object",
          properties: {
            command: { type: "string" },
            workdir: { type: "string" }
          },
          required: ["command", "workdir"]
        }
      },
      {
        name: "apply_patch",
        description: "Apply code changes using unified diff",
        parameters: {
          type: "object",
          properties: {
            patch: { type: "string" }
          },
          required: ["patch"]
        }
      }
      // ... rest
    ]
  }
}
```

---

### 2️⃣ Sandbox (فقط execution environment)

**این container هیچ intelligence نداره - فقط دستورات را اجرا میکنه**

```javascript
class Sandbox {
  constructor(sandboxId, codebasePath) {
    this.id = sandboxId
    this.containerId = null
    this.codebasePath = codebasePath
    this.changes = []
  }

  async create() {
    // ایجاد Docker container
    this.containerId = await docker.createContainer({
      Image: 'sandbox-base:latest',
      WorkingDir: '/workspace',
      HostConfig: {
        Binds: [`${this.codebasePath}:/workspace:rw`],
        Memory: 4 * 1024 * 1024 * 1024, // 4GB
        CpuQuota: 200000, // 2 CPUs
        NetworkMode: 'none' // No network
      }
    })

    await this.containerId.start()
  }

  async executeShell(command, workdir) {
    // فقط اجرای دستور - هیچ تصمیمگیری نداره
    const exec = await this.containerId.exec({
      Cmd: ['sh', '-c', command],
      WorkingDir: workdir,
      AttachStdout: true,
      AttachStderr: true
    })

    const stream = await exec.start()
    
    return new Promise((resolve) => {
      let stdout = '', stderr = ''
      
      docker.modem.demuxStream(
        stream,
        { write: (chunk) => stdout += chunk },
        { write: (chunk) => stderr += chunk }
      )

      stream.on('end', async () => {
        const { ExitCode } = await exec.inspect()
        resolve({ stdout, stderr, exitCode: ExitCode })
      })
    })
  }

  async applyPatch(patchText) {
    // فقط apply کردن patch - بدون reasoning
    const operations = parsePatch(patchText)
    
    for (const op of operations) {
      if (op.type === 'add') {
        await this.writeFile(op.file, op.content)
        this.changes.push({ action: 'add', file: op.file })
      } else if (op.type === 'update') {
        await this.updateFile(op.file, op.hunks)
        this.changes.push({ action: 'modify', file: op.file })
      } // ...
    }

    return { success: true }
  }

  async readFile(path) {
    // فقط خواندن فایل
    const result = await this.executeShell(`cat ${path}`, '/workspace')
    return result.stdout
  }

  async getChanges() {
    return this.changes
  }

  async destroy() {
    await this.containerId.stop()
    await this.containerId.remove()
  }
}
```

---

### 3️⃣ Claude Client (فقط API wrapper)

```javascript
class ClaudeClient {
  constructor(apiKey) {
    this.apiKey = apiKey
    this.baseURL = 'https://openrouter.ai/api/v1'
    this.model = 'anthropic/claude-sonnet-4.5'
  }

  async chat(messages, tools) {
    const response = await fetch(`${this.baseURL}/chat/completions`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        model: this.model,
        messages: messages,
        tools: tools,
        temperature: 0.7,
        max_tokens: 8000
      })
    })

    const data = await response.json()
    const message = data.choices[0].message

    return {
      content: message.content || '',
      toolCalls: (message.tool_calls || []).map(tc => ({
        id: tc.id,
        name: tc.function.name,
        arguments: JSON.parse(tc.function.arguments)
      })),
      finishReason: data.choices[0].finish_reason
    }
  }
}
```

---

### 4️⃣ System Prompt (طبق document شما)

```javascript
function getSystemPrompt() {
  return `You are Codex, an advanced coding agent powered by Claude Sonnet 4.5.

You operate in a sandboxed Docker environment at /workspace.

## Tools Available

**shell**: Execute commands via execvp()
- ALWAYS set workdir parameter
- Use rg (ripgrep) for fast searching
- Commands timeout after 30s

**apply_patch**: Apply code changes using unified diff format
*** Begin Patch ***
Add File: path/to/file.py
+content here

Update File: path/to/existing.py
@@ -1,3 +1,3 @@
-old line
+new line
*** End Patch ***

## Behavior

- Skip planning for simple tasks (~25%)
- For complex tasks: create multi-step plan (2+ steps)
- Use rg or rg --files for searching (faster than grep)
- Apply minimal surgical changes, not full rewrites
- Run tests after significant changes
- Iterate on failures until tests pass
- Report progress concisely

## Completion Criteria

Task is done when:
1. All requirements implemented
2. Tests pass (or created and passing)
3. Code follows project conventions
4. No obvious bugs

Begin by understanding codebase structure, then work systematically.`
}
```

---

### 5️⃣ Frontend Integration

```jsx
function ChatInterface() {
  const [agents, setAgents] = useState([])
  const ws = useWebSocket()

  const startTask = async (task, agentCount, codebase) => {
    // Upload codebase
    const codebaseId = await uploadCodebase(codebase)

    // Start task
    const response = await fetch('/api/tasks/start', {
      method: 'POST',
      body: JSON.stringify({
        task: task,
        agentCount: agentCount,
        codebaseId: codebaseId
      })
    })

    const { taskId, agents: agentIds } = await response.json()

    // Subscribe به agent updates
    agentIds.forEach(id => ws.subscribe(`agent:${id}`))
  }

  useEffect(() => {
    // Real-time updates
    ws.on('agent-reasoning', (data) => {
      // نمایش reasoning
      appendReasoning(data.agentId, data.text)
    })

    ws.on('agent-output', (data) => {
      // نمایش terminal output
      appendTerminal(data.agentId, data.output)
    })

    ws.on('agent-complete', (data) => {
      // نمایش نتیجه
      showResults(data.agentId, data.changes)
    })
  }, [])

  return (
    <div className="flex">
      <AgentTabs agents={agents}>
        {agents.map(agent => (
          <AgentView key={agent.id} agent={agent}>
            <TerminalOutput agentId={agent.id} />
            <ReasoningViewer agentId={agent.id} />
            <FileChanges agentId={agent.id} />
          </AgentView>
        ))}
      </AgentTabs>
    </div>
  )
}
```

---

## تفاوت‌های کلیدی با توضیح قبلی من:

| قبلی (❌ اشتباه) | حالا (✅ درست) |
|------------------|----------------|
| LLM call داخل container | LLM call در orchestrator |
| Agent داخل container | Agent روی host |
| Container = محیط کامل | Container = فقط execution |
| هر agent کامل مستقل | Orchestrator مدیریت میکنه |

---

## Flow دقیق یک iteration:

```
1. User: "Add login feature"
        ↓
2. Orchestrator → Claude: 
   "Here's the task and codebase structure"
        ↓
3. Claude → Orchestrator:
   "I'll use shell to search for auth files"
   tool_call: { name: "shell", args: { command: "rg 'auth'" } }
        ↓
4. Orchestrator → Sandbox:
   "Execute: rg 'auth'"
        ↓
5. Sandbox executes → returns results
        ↓
6. Orchestrator → Claude:
   "Here are the search results: ..."
        ↓
7. Claude → Orchestrator:
   "I'll add login to user.py"
   tool_call: { name: "apply_patch", args: { patch: "..." } }
        ↓
8. Orchestrator → Sandbox:
   "Apply this patch"
        ↓
9. Sandbox applies → returns success
        ↓
10. Orchestrator → Claude:
    "Patch applied successfully"
        ↓
11. Claude: "Now let's test..."
    (loop continues)
```

---

