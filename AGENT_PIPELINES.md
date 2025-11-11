# Документация Пайплайнов Агента Skyvern

> **Автор**: Автоматически сгенерировано
> **Дата**: 2025-11-11
> **Версия**: 1.0

## Оглавление

1. [Введение](#введение)
2. [Task V1 Pipeline - Классическое выполнение](#pipeline-1-task-v1---классическое-выполнение)
3. [Task V2 Pipeline - Агентное планирование](#pipeline-2-task-v2---агентное-планирование)
4. [Workflow Execution Pipeline](#pipeline-3-workflow-execution)
5. [Action Extraction Pipeline](#pipeline-4-action-extraction)
6. [Data Extraction Pipeline](#pipeline-5-data-extraction)
7. [OTP/Authentication Pipeline](#pipeline-6-otpauthentication)
8. [Script Generation Pipeline](#pipeline-7-script-generation)

---

## Введение

Skyvern использует несколько различных пайплайнов для автоматизации браузера. Каждый пайплайн оптимизирован для конкретного типа задач и имеет свою архитектуру выполнения.

### Основные компоненты

- **ForgeAgent** (`skyvern/forge/agent.py`) - главный агент
- **TaskV2Service** (`skyvern/services/task_v2_service.py`) - сервис для Task V2
- **WorkflowService** (`skyvern/forge/sdk/workflow/service.py`) - оркестратор workflow
- **ActionHandler** (`skyvern/webeye/actions/handler.py`) - обработчик действий
- **Scraper** (`skyvern/webeye/scraper/scraper.py`) - скрейпинг страниц

### Общая архитектура данных

```
User Input → Task/Workflow Creation → Browser Session → Execution Loop → Results
                                            ↓
                                    Page Scraping
                                            ↓
                                    LLM Processing
                                            ↓
                                    Action Execution
```

---

## PIPELINE 1: Task V1 - Классическое выполнение

**Назначение**: Пошаговое выполнение задачи с извлечением действий на каждом шаге.

**Entry Point**: `ForgeAgent.execute_step()` → `agent.py:301`

**Когда используется**:
- Простые задачи с четкой последовательностью действий
- Workflow blocks (NavigationBlock, ActionBlock, TaskBlock)
- Задачи созданные через Task V1 API

### Архитектура Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                    TASK V1 EXECUTION FLOW                        │
└─────────────────────────────────────────────────────────────────┘

1. CREATE TASK
   ├─> User Input (navigation_goal, data_extraction_goal)
   ├─> Task creation in database
   └─> Initial Step creation

2. EXECUTE_STEP (Main Loop)
   │
   ├─> Check max_steps limit
   ├─> Check task status (canceled/timeout)
   └─> Call AGENT_STEP
       │
       ├──> 3. BUILD_AND_RECORD_STEP_PROMPT
       │    ├─> Scrape page (DOM + screenshot)
       │    ├─> Build element tree
       │    ├─> Load prompt template (extract-action.j2)
       │    └─> Return: (ScrapedPage, prompt_str)
       │
       ├──> 4. GENERATE ACTIONS (LLM Call)
       │    ├─> Send prompt to LLM
       │    ├─> Parse JSON response
       │    └─> Create Action objects
       │         ├─ CLICK
       │         ├─ INPUT_TEXT
       │         ├─ SELECT_OPTION
       │         ├─ UPLOAD_FILE
       │         ├─ WAIT
       │         ├─ COMPLETE
       │         └─ TERMINATE
       │
       ├──> 5. EXECUTE ACTIONS (ActionHandler)
       │    ├─> For each action:
       │    │   ├─ Validate action
       │    │   ├─ Execute via Playwright
       │    │   └─ Record result
       │    └─> Return: ActionResults
       │
       ├──> 6. COMPLETE VERIFICATION (Optional)
       │    ├─> Check if goal achieved
       │    ├─> Prompt: check-user-goal.j2
       │    └─> Return: user_goal_achieved (bool)
       │
       └──> 7. DECISION
            ├─> If COMPLETE action → End task
            ├─> If TERMINATE action → Fail task
            ├─> If max_steps reached → Fail task
            └─> Else → Create next step, GOTO 2

8. CLEANUP & WEBHOOKS
   ├─> Save artifacts
   ├─> Close browser (if needed)
   └─> Send webhook notification
```

### Детальный flow по методам

#### 1. `execute_step()` - Main orchestrator
```
Input:
  - task: Task
  - step: Step
  - organization: Organization
  - engine: RunEngine (default: skyvern_v1)

Flow:
  1. Set context (step_id, task_id)
  2. Check workflow_run status (canceled/timeout)
  3. Check task status (canceled)
  4. Validate max_steps limit
  5. Initialize browser_state
  6. Call agent_step() [LOOP до COMPLETE/TERMINATE]
  7. Handle step completion/failure
  8. Cleanup & webhooks

Output:
  - step: Step (final)
  - detailed_output: DetailedAgentStepOutput
  - next_step: Step | None
```

#### 2. `agent_step()` - Single step execution
```
Input:
  - task: Task
  - step: Step
  - browser_state: BrowserState
  - engine: RunEngine

Flow:
  1. Update step status → RUNNING
  2. Call build_and_record_step_prompt()
     → Returns (ScrapedPage, prompt, use_caching)

  3. Generate actions based on engine:
     IF engine == skyvern_v1:
       - Call LLM with extract-action prompt
       - Parse JSON → List[Action]

     ELIF engine == openai_cua:
       - Call _generate_cua_actions()

     ELIF engine == anthropic_cua:
       - Call _generate_anthropic_actions()

  4. Execute actions via ActionHandler
     - For each action in actions:
       - handler.handle_action(action, page)

  5. Complete verification (if enabled):
     - Call check_user_goal_complete()
     - Prompt: check-user-goal.j2

  6. Update step status → COMPLETED/FAILED

Output:
  - step: Step (updated)
  - detailed_output: DetailedAgentStepOutput
```

#### 3. `build_and_record_step_prompt()` - Scraping & Prompt Building
```
Input:
  - task: Task
  - step: Step
  - browser_state: BrowserState

Flow:
  1. Get current page from browser_state

  2. Scrape page:
     scrape_website(
       browser_state,
       url=current_url,
       scrape_types=[ScrapeType.HTML, ScrapeType.ACCESSIBILITY_TREE]
     )
     → Returns ScrapedPage

  3. Take screenshot:
     - Full page screenshot
     - Save as artifact

  4. Build element tree:
     - Parse DOM
     - Extract interactable elements
     - Assign unique IDs

  5. Build prompt:
     IF task has navigation_goal:
       - Load template: extract-action.j2
       - Variables:
         - navigation_goal: task.navigation_goal
         - elements: scraped_page.elements
         - current_url: page.url
         - action_history: get_action_history(task)
         - navigation_payload_str: task.navigation_payload

     ELIF task has only data_extraction_goal:
       - Create ExtractAction directly

  6. Save prompt artifact

Output:
  - scraped_page: ScrapedPage
  - prompt: str
  - use_caching: bool
```

### Промпты и их последовательность

#### Основной цикл (каждый шаг):

**1. Extract Actions** (`extract-action.j2`)
```
Input variables:
  - {{ navigation_goal }}
  - {{ elements }}
  - {{ current_url }}
  - {{ action_history }}
  - {{ navigation_payload_str }}
  - {{ data_extraction_goal }} (optional)
  - {{ complete_criterion }} (optional)

Output schema:
{
  "user_goal_stage": str,
  "user_goal_achieved": bool,
  "action_plan": str,
  "actions": [{
    "action_type": "CLICK" | "INPUT_TEXT" | "SELECT_OPTION" | ...,
    "id": str,
    "reasoning": str,
    "confidence_float": float,
    ...
  }]
}
```

**2. Check Goal Complete** (`check-user-goal.j2`) - если complete_verification=True
```
Input variables:
  - {{ navigation_goal }}
  - {{ elements }}
  - {{ navigation_payload }}
  - {{ complete_criterion }} (optional)

Output schema:
{
  "page_info": str,
  "thoughts": str,
  "user_goal_achieved": bool
}
```

**3. Extract Information** (`extract-information.j2`) - если есть data_extraction_goal
```
Input variables:
  - {{ data_extraction_goal }}
  - {{ extracted_information_schema }}
  - {{ elements }}
  - {{ current_url }}
  - {{ extracted_text }}

Output schema:
  Defined by extracted_information_schema (user-provided JSON schema)
```

### Передача данных между шагами

```
Step N:
  ├─> ScrapedPage.elements → extract-action prompt
  ├─> LLM Response → Actions
  ├─> Actions → ActionResults
  └─> ActionResults → Saved to database

Step N+1:
  ├─> Loads action_history from database
  ├─> action_history → extract-action prompt
  └─> LLM uses history to avoid repeated actions
```

### Exit условия

1. **Success - COMPLETE action**
   - LLM returns action_type="COMPLETE"
   - user_goal_achieved=true
   - Task status → COMPLETED

2. **Failure - TERMINATE action**
   - LLM returns action_type="TERMINATE"
   - Task deemed impossible
   - Task status → FAILED

3. **Failure - Max Steps**
   - step.order >= max_steps_per_run
   - Call summarize-max-steps-reason.j2
   - Task status → FAILED

4. **Failure - Exception**
   - Browser crash, timeout, error
   - Task status → FAILED

### Пример выполнения

```
Task: "Login to gmail.com with credentials and check inbox"
Max Steps: 10

Step 1:
  Scrape → gmail.com login page
  Prompt → extract-action.j2
  LLM → [INPUT_TEXT(email_field), CLICK(next_button)]
  Execute → Enter email, click next
  Result → Success

Step 2:
  Scrape → password page
  Prompt → extract-action.j2 (with history from Step 1)
  LLM → [INPUT_TEXT(password_field), CLICK(login_button)]
  Execute → Enter password, click login
  Result → Success

Step 3:
  Scrape → inbox page
  Prompt → extract-action.j2
  LLM → [COMPLETE]
  Execute → Mark task complete
  Result → TASK COMPLETED
```

---

