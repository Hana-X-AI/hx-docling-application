# Execute Sprint Tasks

Execute tasks from a sprint in the project task directory.

## Arguments

Format: `$ARGUMENTS` should be one of:
- `list` - Show all available sprints
- `<sprint-id>` - Show tasks in a specific sprint (e.g., "1.1", "1.5a")
- `<sprint-id> <task-id> <agent>` - Execute a specific task with a specific agent

## Instructions

You are executing sprint tasks from the `project/0.4-tasks/` directory.

Parse `$ARGUMENTS` to determine the action:

---

### Mode 1: List All Sprints (`/sprint list` or `/sprint`)

If arguments is "list" or empty, list all available sprints by reading the sprint directories:

```
project/0.4-tasks/sprint-*/
```

Show each sprint with:
- Sprint number and name (from directory name)
- Number of task files
- Agents involved (from filenames like `neo-nextjs-tasks.md`)

---

### Mode 2: Show Sprint Details (`/sprint <sprint-id>`)

If only one argument is provided (e.g., `/sprint 1.1`):

1. **Find the sprint directory** matching the pattern `sprint-<sprint-id>*` or `sprint-<sprint-id>-*`

2. **List all task files** in that sprint directory (files ending in `-tasks.md`)

3. **For each task file**, extract and display:
   - Agent name (from filename)
   - Task IDs and titles (from task headers like `### NEO-1.1-001:`)
   - Priority and effort for each task

4. **Present a summary** in table format showing:
   - Task ID
   - Task Title
   - Agent
   - Priority
   - Effort

---

### Mode 3: Execute Task (`/sprint <sprint-id> <task-id> <agent>`)

If three arguments are provided (e.g., `/sprint 1.1 NEO-1.1-001 neo`):

1. **Parse arguments**:
   - Sprint ID: First argument (e.g., "1.1")
   - Task ID: Second argument (e.g., "NEO-1.1-001")
   - Agent: Third argument (e.g., "neo")

2. **Find the task file** in `project/0.4-tasks/sprint-<sprint-id>*/` that contains the task ID

3. **Read the task file** and extract the specific task section matching the task ID

4. **Execute the task** using the Task tool:
   ```
   Task tool with:
   - subagent_type: <agent>
   - prompt: Include the full task details (description, acceptance criteria, deliverables, technical notes)
   - description: "Execute <task-id>"
   ```

5. **Track progress** using TodoWrite with the task details

---

## Agent Reference

Valid agent values for the third argument:

| Agent | Subagent Type | Specialty |
|-------|---------------|-----------|
| Neo | `neo` | Next.js development |
| Bob | `bob` | FastAPI backend |
| Trinity | `trinity` | PostgreSQL DBA |
| Sri | `sri` | Redis caching |
| George | `george` | FastMCP gateway |
| James | `james` | Docling MCP |
| Gordon | `gordon` | shadcn/ui components |
| Ola | `ola` | Frontend UI (Thesys C1) |
| Julia | `julia` | Testing & QA |
| William | `william` | Infrastructure ops |
| Sophia | `sophia` | LangGraph orchestration |
| Amanda | `amanda` | Ansible automation |
| Albert | `albert` | Docling document processing |
| Paul | `paul` | Pydantic validation |
| Thomas | `thomas` | Docker containerization |
| Shane | `shane` | LiteLLM proxy |
| Mitch | `mitch` | Qdrant vector DB |

---

## Examples

```bash
# List all sprints
/sprint list

# Show Sprint 1.1 tasks and details
/sprint 1.1

# Execute a specific task with Neo agent
/sprint 1.1 NEO-1.1-001 neo

# Execute a PostgreSQL task with Trinity agent
/sprint 1.1 TRINITY-1.1-001 trinity

# Execute a testing task with Julia agent
/sprint 1.1 JULIA-1.1-001 julia
```

---

## Task Execution Workflow

When executing a task (`/sprint <sprint-id> <task-id> <agent>`):

1. Create a todo item for the task (in_progress)
2. Read the full task details from the task file
3. Invoke the Task tool with the appropriate agent
4. The agent will:
   - Review the task requirements
   - Implement the deliverables
   - Verify acceptance criteria
5. Mark the todo as completed when done
6. Report the results back to the user
