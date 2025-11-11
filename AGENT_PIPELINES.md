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


## PIPELINE 2: Task V2 - Агентное планирование

**Назначение**: Итеративное планирование задач с LLM-управляемым выбором стратегии на каждой итерации.

**Entry Point**: `run_task_v2()` → `task_v2_service.py:356`

**Когда используется**:
- Сложные задачи, требующие динамического планирования
- Задачи без четкой последовательности действий
- API endpoint: `/api/v2/task_v2` 
- Workflow blocks с Task V2

### Ключевые отличия от Task V1

| Аспект | Task V1 | Task V2 |
|--------|---------|---------|
| **Планирование** | Извлекает все действия сразу на шаг | LLM планирует по одной задаче за раз |
| **Стратегия** | Фиксированная (action extraction) | Динамическая (navigate/extract/loop) |
| **Итерации** | Max steps (обычно 10-20) | Max iterations (50) |
| **Гибкость** | Низкая | Высокая - LLM решает стратегию |
| **Use case** | Простые последовательные задачи | Сложные исследовательские задачи |

### Архитектура Pipeline

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      TASK V2 EXECUTION FLOW                               │
└──────────────────────────────────────────────────────────────────────────┘

1. INITIALIZATION (initialize_task_v2)
   ├─> Create TaskV2 in database
   ├─> Create auto-generated Workflow
   ├─> Create WorkflowRun
   └─> Call task_v2_generate_metadata prompt
       ├─ Input: user_prompt, url
       ├─ Output: { "title": str, "url": str }
       └─ Update TaskV2 metadata

2. MAIN PLANNING LOOP (max 50 iterations)
   │
   ┌──> Iteration N:
   │   │
   │   ├──> 3. SCRAPE PAGE
   │   │    ├─> scrape_website(browser_state, current_url)
   │   │    └─> Returns: ScrapedPage (elements, screenshots)
   │   │
   │   ├──> 4. PLANNING CALL (task_v2 prompt)
   │   │    ├─> Load prompt: task_v2.j2
   │   │    ├─> Variables:
   │   │    │   - user_goal: task_v2.prompt
   │   │    │   - current_url: current page URL
   │   │    │   - elements: scraped_page.elements
   │   │    │   - task_history: list of completed tasks
   │   │    ├─> LLM Call → JSON response
   │   │    └─> Parse:
   │   │        {
   │   │          "page_info": str,          // Observation
   │   │          "extraction_thought": str, // Should extract?
   │   │          "require_extraction": bool,
   │   │          "thoughts": str,           // Reasoning
   │   │          "user_goal_achieved": bool,// Done?
   │   │          "plan": str,               // Next mini-goal
   │   │          "task_type": str,          // navigate|extract|loop
   │   │          "loop_values": list[str],  // For loop type
   │   │        }
   │   │
   │   ├──> 5. DECISION POINT
   │   │    ├─> IF user_goal_achieved == true:
   │   │    │   ├─ Call _summarize_task_v2()
   │   │    │   │  └─ Prompt: task_v2_summary.j2
   │   │    │   └─ BREAK loop → Task completed
   │   │    │
   │   │    └─> ELSE: Generate and execute task block
   │   │
   │   ├──> 6. GENERATE TASK BLOCK (based on task_type)
   │   │    │
   │   │    ├─> IF task_type == "navigate":
   │   │    │   ├─ Create NavigationBlock
   │   │    │   ├─ navigation_goal = mini_goal template
   │   │    │   └─ Calls Task V1 pipeline internally
   │   │    │
   │   │    ├─> IF task_type == "extract":
   │   │    │   ├─ Call _generate_extraction_task()
   │   │    │   ├─ Prompt: task_v2_generate_extraction_task.j2
   │   │    │   │  └─ Generates JSON schema for extraction
   │   │    │   ├─ Create ExtractionBlock
   │   │    │   └─ Calls Data Extraction pipeline
   │   │    │
   │   │    └─> IF task_type == "loop":
   │   │        ├─ Call _generate_loop_task()
   │   │        ├─ Prompt: task_v2_loop_task_extraction_goal.j2
   │   │        │  └─ Extracts loop values from page
   │   │        ├─ Create ForLoopBlock
   │   │        │  └─ Contains inner task blocks
   │   │        └─ Execute loop iterations
   │   │
   │   ├──> 7. EXECUTE BLOCK
   │   │    ├─> Call workflow_service.execute_block()
   │   │    ├─> Block executes (may call Task V1 internally)
   │   │    └─> Returns: BlockResult
   │   │
   │   ├──> 8. RECORD HISTORY
   │   │    ├─> task_history.append({
   │   │    │     "type": task_type,
   │   │    │     "task": plan,
   │   │    │     "status": block_result.status
   │   │    │   })
   │   │    └─> History used in next iteration
   │   │
   │   └──> 9. UPDATE WORKFLOW
   │        ├─> Add block_yaml to workflow definition
   │        ├─> Add parameters to workflow
   │        └─> Update current_url from page
   │
   └──> LOOP BACK to step 3 (Iteration N+1)

10. COMPLETION CHECK (after loop or break)
    ├─> Call task_v2_check_completion.j2
    ├─> Mark TaskV2 as completed/failed
    └─> Cleanup browser (if needed)

11. FAILURE HANDLING (if max iterations reached)
    ├─> Call _summarize_max_steps_failure_reason()
    ├─> Prompt: task_v2_summarize-max-steps-reason.j2
    └─> Mark TaskV2 as failed
```

### Детальный flow по методам

#### 1. `initialize_task_v2()` - Task creation
```
Input:
  - user_prompt: str
  - user_url: str | None
  - organization: Organization
  - extracted_information_schema: dict | None

Flow:
  1. Create TaskV2 in database
  2. Create auto-generated Workflow
  3. Create WorkflowRun (status=queued)
  4. Link TaskV2 ↔ WorkflowRun
  5. Set context (task_v2_id, workflow_run_id)

Output:
  - task_v2: TaskV2
```

#### 2. `initialize_task_v2_metadata()` - Generate metadata
```
Input:
  - task_v2: TaskV2
  - user_prompt: str
  - current_browser_url: str | None
  - user_url: str | None

Flow:
  1. Scrape current page (if exists)

  2. Build prompt: task_v2_generate_metadata.j2
     Variables:
       - {{ user_prompt }}
       - {{ current_browser_url }}
       - {{ user_url }}

  3. LLM Call → { "title": str, "url": str }

  4. Update TaskV2:
     - task_v2.title = response["title"]
     - task_v2.url = response["url"]

Output:
  - task_v2: TaskV2 (with title and url)
```

#### 3. `run_task_v2_helper()` - Main execution loop
```
Input:
  - task_v2: TaskV2
  - organization: Organization

Flow:
  1. Get or create browser_state
  2. Initialize variables:
     - task_history = []
     - yaml_blocks = []
     - current_url = page.url

  3. FOR i in range(DEFAULT_MAX_ITERATIONS=50):
     
     a) Validate task (check cancel/timeout)
     
     b) IF i == 0 (first iteration):
        - Navigate to task_v2.url
        - Create goto_url block
     
     ELSE:
        c) Scrape current page
        
        d) Call task_v2 planning prompt
           prompt = load_prompt_with_elements(
             scraped_page,
             "task_v2",
             user_goal=task_v2.prompt,
             task_history=task_history
           )
           
        e) LLM Call → task_v2_response
        
        f) Parse response:
           - user_goal_achieved
           - task_type (navigate/extract/loop)
           - plan (mini-goal)
        
        g) IF user_goal_achieved:
           - Call _summarize_task_v2()
           - BREAK loop
        
        h) Generate block based on task_type:
           - navigate → NavigationBlock
           - extract → ExtractionBlock  
           - loop → ForLoopBlock
        
        i) Execute block via workflow_service
        
        j) Record to task_history
        
        k) Update current_url from page

  4. IF loop completed without break:
     - Reached max iterations
     - Call _summarize_max_steps_failure_reason()
     - Mark task as failed

  5. Cleanup & webhooks

Output:
  - task_v2: TaskV2 (completed/failed)
  - workflow: Workflow (with generated blocks)
  - workflow_run: WorkflowRun
```

### Промпты и их последовательность

#### Инициализация:

**1. Generate Metadata** (`task_v2_generate_metadata.j2`)
```
Input:
  - {{ user_prompt }}
  - {{ current_browser_url }}
  - {{ user_url }}

Output:
{
  "title": str,  // Task title
  "url": str     // Resolved starting URL
}

Вызывается: 1 раз при создании Task V2
```

#### Основной цикл (каждая итерация):

**2. Planning** (`task_v2.j2`) - ❗ КЛЮЧЕВОЙ ПРОМПТ
```
Input:
  - {{ user_goal }}      // Исходная цель пользователя
  - {{ current_url }}    // Текущий URL страницы
  - {{ elements }}       // DOM элементы
  - {{ task_history }}   // История выполненных задач
  - {{ local_datetime }} // Текущее время

Output:
{
  "page_info": str,                  // Описание страницы
  "extraction_thought": str,         // Нужно ли извлекать данные?
  "require_extraction": bool,
  "task_history_information": str,   // Что уже сделано
  "information_extracted": bool|null,
  "thoughts": str,                   // Рассуждения (step-by-step)
  "user_goal_achieved": bool,        // Цель достигнута?
  "plan": str,                       // Следующая мини-цель
  "task_type": "navigate"|"extract"|"loop",
  "loop_values": list[str]|null,     // Для loop типа
  "is_loop_value_link": bool         // URL ли значения loop
}

Вызывается: Каждую итерацию (до 50 раз)
```

#### Генерация задач (по типу):

**3a. Generate Extraction Task** (`task_v2_generate_extraction_task.j2`)
```
Input:
  - {{ data_extraction_goal }}
  - {{ extracted_information_schema }} (если задана)
  - {{ elements }}
  - {{ task_history }}

Output:
{
  "thought": str,
  "suggested_data_schema": {...}  // JSON schema для extraction
}

Вызывается: Когда task_type == "extract"
```

**3b. Loop Task Extraction Goal** (`task_v2_loop_task_extraction_goal.j2`)
```
Input:
  - {{ natural_language_prompt }}  // План для loop
  - {{ elements }}
  - Generated schema с loop_values

Output:
{
  "loop_values": [str, str, ...],     // Список значений для итерации
  "is_loop_value_link": bool
}

Вызывается: Когда task_type == "loop"
```

#### Завершение:

**4. Task V2 Summary** (`task_v2_summary.j2`)
```
Input:
  - {{ user_goal }}
  - {{ task_history }}
  - {{ current_url }}
  - Screenshots

Output:
{
  "summary": str,  // Итоговое резюме выполнения
  "status": str    // Success/Partial/Failed
}

Вызывается: При user_goal_achieved=true
```

**5. Check Completion** (`task_v2_check_completion.j2`)
```
Input:
  - {{ user_goal }}
  - {{ current_url }}
  - {{ elements }}
  - {{ task_history }}

Output:
{
  "reasoning": str,
  "user_goal_achieved": bool
}

Вызывается: Для финальной проверки
```

**6. Summarize Max Steps Reason** (`task_v2_summarize-max-steps-reason.j2`)
```
Input:
  - {{ navigation_goal }}
  - {{ history }}  // Block history
  - {{ block_cnt }}

Output:
{
  "page_info": str,
  "reasoning": str  // Почему не достигли цели
}

Вызывается: При достижении max iterations
```

### Передача данных между итерациями

```
Iteration 1:
  Scrape → task_v2 prompt → "navigate to login"
  Execute → NavigationBlock
  Result → {type: "navigate", task: "login", status: "completed"}
  └─> Added to task_history

Iteration 2:
  task_history = [iteration 1 result]
  Scrape → task_v2 prompt (with history) → "extract user info"
  Execute → ExtractionBlock
  Result → {type: "extract", task: "get user data", status: "completed"}
  └─> Added to task_history

Iteration 3:
  task_history = [iteration 1, iteration 2]
  Scrape → task_v2 prompt (with history) → user_goal_achieved=true
  └─> Summarize and complete
```

### Mini-Goal Template

Когда LLM возвращает task_type="navigate", создается NavigationBlock с навигационной целью:

```python
MINI_GOAL_TEMPLATE = """Achieve the following mini goal and once it's achieved, complete:
```{mini_goal}```

This mini goal is part of the big goal the user wants to achieve and use the big goal as context to achieve the mini goal:
```{main_goal}```"""
```

Это помогает Task V1 (внутри NavigationBlock) понять контекст и остановиться после достижения мини-цели.

### Exit условия

1. **Success - user_goal_achieved=true**
   - LLM определяет, что цель достигнута
   - task_v2_response["user_goal_achieved"] == true
   - Call _summarize_task_v2()
   - TaskV2 status → COMPLETED

2. **Failure - Max Iterations**
   - i >= DEFAULT_MAX_ITERATIONS (50)
   - Call _summarize_max_steps_failure_reason()
   - TaskV2 status → FAILED

3. **Failure - Workflow Canceled**
   - workflow_run.status == CANCELED
   - TaskV2 status → CANCELED

4. **Failure - Exception**
   - Browser crash, LLM error, etc.
   - TaskV2 status → FAILED

### Пример выполнения

```
Task V2: "Find the top 5 posts on Hacker News and extract their titles and links"

Iteration 1:
  Planning → "Go to Hacker News homepage"
  Type → navigate
  Execute → NavigationBlock (goes to news.ycombinator.com)
  Result → Success

Iteration 2:
  Planning → "Extract the list of top posts with their titles and links"
  Type → loop
  Sub-prompts:
    - task_v2_loop_task_extraction_goal → Extracts 5 post links
    - task_v2_generate_extraction_task → Creates schema for title+link
  Execute → ForLoopBlock
    - Inner ExtractionBlock for each link
  Result → Extracted 5 posts

Iteration 3:
  Planning → user_goal_achieved=true (all data extracted)
  Summarize → "Successfully extracted 5 posts from HN"
  Result → COMPLETED
```

### Особенности реализации

1. **Workflow Auto-generation**
   - Task V2 динамически создает Workflow
   - Каждая итерация добавляет блоки в YAML
   - Финальный workflow можно переиспользовать

2. **Block Nesting**
   - NavigationBlock вызывает Task V1 pipeline
   - ExtractionBlock вызывает Data Extraction pipeline
   - ForLoopBlock рекурсивно выполняет inner blocks

3. **Context Awareness**
   - task_history хранит всю историю
   - LLM видит, что уже сделано
   - Избегает повторяющихся действий

4. **Flexibility**
   - LLM сам выбирает стратегию
   - Может переключаться navigate ↔ extract
   - Адаптируется к изменениям на странице

---


## PIPELINE 3: Workflow Execution

**Назначение**: Оркестрация последовательного выполнения блоков workflow с передачей параметров.

**Entry Point**: `WorkflowService.execute_workflow()` → `workflow/service.py`

**Когда используется**:
- Workflow созданные через YAML/JSON API
- Многошаговые автоматизации с разными типами блоков
- Переиспользуемые процессы

### Архитектура

```
┌────────────────────────────────────────────────────────────┐
│             WORKFLOW EXECUTION FLOW                         │
└────────────────────────────────────────────────────────────┘

Workflow Definition (YAML)
  ├─> parameters: [param1, param2, ...]
  └─> blocks: [block1, block2, ...]

EXECUTION:

1. CREATE WORKFLOW RUN
   ├─> Status: queued
   ├─> Initialize context with parameters
   └─> Resolve Jinja2 templates

2. FOR EACH BLOCK (sequential):
   │
   ├──> 3. PRE-EXECUTION
   │    ├─> Resolve block parameters from context
   │    ├─> Create WorkflowRunBlock (status=running)
   │    └─> Set up block-specific requirements
   │
   ├──> 4. EXECUTE BLOCK (type-specific)
   │    │
   │    ├─> TaskBlock/NavigationBlock:
   │    │   └─> Calls Task V1 pipeline
   │    │
   │    ├─> ExtractionBlock:
   │    │   └─> Calls Data Extraction pipeline
   │    │
   │    ├─> ForLoopBlock:
   │    │   ├─> Extract loop values
   │    │   ├─> FOR value in loop_values:
   │    │   │   └─> Recursively execute loop_blocks
   │    │   └─> Aggregate results
   │    │
   │    ├─> ValidationBlock:
   │    │   ├─> Check complete_criterion OR
   │    │   └─> Check terminate_criterion
   │    │
   │    ├─> CodeBlock:
   │    │   └─> Execute Python in sandbox
   │    │
   │    ├─> SendEmailBlock:
   │    │   └─> Send via SMTP
   │    │
   │    ├─> DownloadToS3Block:
   │    │   └─> Upload files to S3
   │    │
   │    ├─> UploadToS3Block:
   │    │   └─> Upload to S3 from URL
   │    │
   │    ├─> FileParserBlock:
   │    │   └─> Parse CSV/PDF/DOCX
   │    │
   │    ├─> TextPromptBlock:
   │    │   └─> LLM text generation
   │    │
   │    └─> WaitBlock:
   │        └─> Sleep(seconds)
   │
   ├──> 5. POST-EXECUTION
   │    ├─> Extract output parameters
   │    ├─> Update context with outputs
   │    ├─> Mark block as completed/failed
   │    └─> Save BlockResult
   │
   └──> 6. CONTINUE/TERMINATE
        ├─> If block failed AND continue_on_failure=false:
        │   └─> Stop workflow → FAILED
        │
        ├─> If ValidationBlock triggered terminate:
        │   └─> Stop workflow → FAILED
        │
        └─> Else: Next block

7. FINALIZATION
   ├─> Mark WorkflowRun as completed/failed
   ├─> Cleanup browser
   └─> Execute webhooks
```

### Типы блоков и их промпты

| Block Type | Calls Pipeline | Key Prompts |
|------------|----------------|-------------|
| TaskBlock | Task V1 | extract-action.j2, check-user-goal.j2 |
| NavigationBlock | Task V1 | extract-action.j2 |
| ExtractionBlock | Data Extraction | extract-information.j2 |
| ActionBlock | Task V1 | extract-action.j2 |
| FileDownloadBlock | Task V1 | extract-action.j2 |
| ValidationBlock | Validation | decisive-criterion-validate.j2 |
| ForLoopBlock | Recursive | extraction_prompt_for_nat_language_loops.j2 |
| TextPromptBlock | LLM Direct | (user-provided prompt) |
| CodeBlock | Python Exec | (no prompt) |

### Пример Workflow

```yaml
parameters:
  - key: product_name
    workflow_parameter_type: string
  - key: results
    workflow_parameter_type: json

blocks:
  - block_type: navigation
    label: "Search for product"
    url: "https://amazon.com"
    navigation_goal: "Search for {{ product_name }}"
    
  - block_type: extraction
    label: "Extract price"
    data_extraction_goal: "Get product price"
    data_schema: {"price": {"type": "string"}}
    output_parameter_key: "results"
    
  - block_type: code
    label: "Process results"
    code: |
      output = {"product": product_name, "price": results["price"]}
```

### Context Parameter Passing

```
Block 1 (Extraction):
  Output: results = {"price": "$99.99"}
  └─> context["results"] = {"price": "$99.99"}

Block 2 (Code):
  Input: {{ results.price }}  # Jinja2 resolves to "$99.99"
  Output: processed_data = {"final_price": 99.99}
  └─> context["processed_data"] = {"final_price": 99.99}

Block 3 (SendEmail):
  Input: body = "Price is {{ processed_data.final_price }}"
  └─> Sends email: "Price is 99.99"
```

---

## PIPELINE 4: Action Extraction

**Назначение**: Извлечение и выполнение действий из скрейпленной страницы.

**Entry Point**: `agent_step()` → `agent.py:876`

**Является частью**: Task V1 pipeline (вызывается каждый шаг)

### Архитектура

```
┌────────────────────────────────────────────────────────────┐
│             ACTION EXTRACTION FLOW                          │
└────────────────────────────────────────────────────────────┘

1. SCRAPE PAGE
   ├─> scrape_website(browser_state, url)
   ├─> Extract DOM tree
   ├─> Parse HTML elements
   ├─> Extract accessibility tree
   ├─> Assign unique IDs to interactable elements
   └─> ScrapedPage:
       ├─ elements: List[dict] # {id, tag, text, attributes}
       ├─ element_tree: str
       ├─ screenshots: List[bytes]
       └─ url: str

2. BUILD PROMPT
   ├─> Load template: extract-action.j2
   ├─> Inject variables:
   │   - navigation_goal
   │   - elements (filtered, ID-tagged)
   │   - current_url
   │   - action_history
   │   - navigation_payload
   │   - screenshots (attached to LLM call)
   └─> Full prompt string

3. LLM CALL (Engine-specific)
   │
   ├─> IF engine == skyvern_v1:
   │   ├─> Call LLM_API_HANDLER(prompt, screenshots)
   │   └─> Returns: JSON string
   │
   ├─> IF engine == openai_cua:
   │   ├─> _generate_cua_actions()
   │   └─> Returns: Action objects directly
   │
   ├─> IF engine == anthropic_cua:
   │   ├─> _generate_anthropic_actions()
   │   └─> Returns: Action objects
   │
   └─> IF engine == ui_tars:
       ├─> _generate_ui_tars_actions()
       ├─> Prompt: ui-tars-system-prompt.j2
       └─> Returns: Coordinate-based actions

4. PARSE ACTIONS (for skyvern_v1)
   ├─> parse_actions(json_response)
   ├─> Validate JSON schema
   ├─> Create Action objects:
   │   - ClickAction(element_id, ...)
   │   - InputTextAction(element_id, text, ...)
   │   - SelectOptionAction(element_id, option, ...)
   │   - UploadFileAction(element_id, file_url, ...)
   │   - WaitAction()
   │   - CompleteAction()
   │   - TerminateAction()
   └─> List[Action]

5. EXECUTE ACTIONS (ActionHandler)
   │
   FOR action in actions:
       │
       ├──> Validate action
       │    ├─> Check element still exists
       │    └─> Check element is interactable
       │
       ├──> Execute via Playwright
       │    ├─> ClickAction → page.click(selector)
       │    ├─> InputTextAction → page.fill(selector, text)
       │    ├─> SelectOptionAction → page.select_option(...)
       │    ├─> UploadFileAction → page.set_input_files(...)
       │    └─> WaitAction → page.wait_for_timeout(...)
       │
       ├──> Record result
       │    ├─> ActionSuccess(action, ...)
       │    └─> ActionFailure(action, exception, ...)
       │
       └──> Continue to next action

6. RETURN RESULTS
   └─> List[ActionResult]
```

### Action Types Detail

```python
# Основные типы действий:

CLICK:           # Клик по элементу
  - element_id
  - download: bool  # Триггерит ли загрузку
  - file_url: str   # Для file upload через клик

INPUT_TEXT:      # Ввод текста
  - element_id
  - text: str

SELECT_OPTION:   # Выбор из dropdown
  - element_id
  - option: {label, value, index}

UPLOAD_FILE:     # Загрузка файла
  - element_id
  - file_url: str

WAIT:            # Ожидание
  - No parameters

COMPLETE:        # Цель достигнута
  - Ends task successfully

TERMINATE:       # Невозможно выполнить
  - Ends task with failure
```

### Element Selection Process

```
HTML Page:
  <button id="abc123" class="btn">Submit</button>

Scraper:
  └─> Assigns unique ID: "skyvern_id_001"

ScrapedPage.elements:
  [{
    "id": "skyvern_id_001",
    "tag": "button",
    "text": "Submit",
    "attributes": {"class": "btn"},
    "interactable": true
  }]

LLM Prompt:
  "...elements from page:
   [id=skyvern_id_001, tag=button, text=Submit]"

LLM Response:
  {"action_type": "CLICK", "id": "skyvern_id_001"}

ActionHandler:
  └─> Resolves "skyvern_id_001" → actual DOM element
  └─> page.click(selector_for_abc123)
```

---

## PIPELINE 5: Data Extraction

**Назначение**: Извлечение структурированных данных из страницы согласно JSON schema.

**Entry Point**: 
- `create_extract_action()` → `agent.py`
- `ExtractionBlock.execute()` → `workflow/models/block.py`

### Архитектура

```
┌────────────────────────────────────────────────────────────┐
│             DATA EXTRACTION FLOW                            │
└────────────────────────────────────────────────────────────┘

1. PREPARATION
   ├─> User provides:
   │   - data_extraction_goal: str
   │   - extracted_information_schema: dict (JSON Schema)
   ├─> Scrape current page
   └─> Extract text content

2. BUILD PROMPT
   ├─> Load template: extract-information.j2
   ├─> Variables:
   │   - {{ data_extraction_goal }}
   │   - {{ extracted_information_schema }}
   │   - {{ elements }}
   │   - {{ extracted_text }}
   │   - {{ current_url }}
   └─> Attach screenshots

3. LLM CALL
   ├─> LLM_API_HANDLER(prompt, screenshots)
   └─> Returns: JSON matching schema

4. VALIDATE OUTPUT
   ├─> Check JSON validity
   ├─> Validate against schema
   ├─> Check required fields
   └─> Null for missing data

5. RETURN
   └─> Extracted data (dict/list)
```

### Schema Example

```json
Input Schema:
{
  "products": {
    "type": "array",
    "items": {
      "type": "object",
      "properties": {
        "name": {"type": "string"},
        "price": {"type": "string"},
        "rating": {"type": "number"}
      },
      "required": ["name", "price"]
    }
  }
}

LLM Output:
{
  "products": [
    {"name": "iPhone 15", "price": "$999", "rating": 4.5},
    {"name": "Samsung Galaxy", "price": "$849", "rating": null}
  ]
}
```

---

## PIPELINE 6: OTP/Authentication

**Назначение**: Автоматическое получение и ввод OTP кодов для верификации.

**Entry Point**: 
- `poll_otp_value()` → `services/otp_service.py`
- Triggered during Task V1 execution

### Архитектура

```
┌────────────────────────────────────────────────────────────┐
│             OTP AUTHENTICATION FLOW                         │
└────────────────────────────────────────────────────────────┘

1. DETECTION
   ├─> During action extraction
   ├─> LLM identifies verification code field
   └─> Returns: should_enter_verification_code=true

2. POLLING (poll_otp_value)
   │
   ├──> Check task.totp_identifier
   │    └─> Bitwarden TOTP key / Email address
   │
   ├──> IF Bitwarden TOTP:
   │    ├─> Connect to Bitwarden
   │    ├─> Fetch TOTP secret
   │    └─> Generate current code
   │
   └──> IF Email/SMS (via totp_verification_url):
       │
       ├──> 3. FETCH MESSAGE
       │    ├─> Call external OTP service
       │    └─> Get email/SMS content
       │
       ├──> 4. PARSE OTP
       │    ├─> Load prompt: parse-otp-login.j2
       │    ├─> Variables: {{ content }}
       │    ├─> LLM extracts:
       │    │   - otp_type: "totp" | "magic_link"
       │    │   - otp_value: str
       │    └─> Returns OTP code/link
       │
       └──> 5. RETURN CODE
            └─> otp_value: str

6. INPUT CODE
   ├─> Create INPUT_TEXT action
   ├─> Fill verification code field
   └─> Continue task execution
```

### OTP Sources

1. **Bitwarden TOTP**
   - Stored TOTP secrets
   - Real-time code generation
   - No external API needed

2. **Email Polling**
   - External service (totp_verification_url)
   - Periodically checks inbox
   - Extracts code via parse-otp-login.j2

3. **Magic Links**
   - Extracted from email
   - Opens link in browser
   - Continues navigation

### Example Flow

```
Task: Login to gmail.com

Step 1: Enter email → Next
Step 2: Enter password → Next
Step 3: 
  ├─> Page shows "Enter verification code"
  ├─> LLM detects verification_code field
  └─> should_enter_verification_code=true

OTP Pipeline Triggered:
  ├─> Check totp_identifier="user@example.com"
  ├─> Poll external service for email
  ├─> Receive email: "Your code is 123456"
  ├─> parse-otp-login prompt → otp_value="123456"
  └─> Return code

Step 3 (continued):
  ├─> INPUT_TEXT(verification_field, "123456")
  └─> CLICK(verify_button)

Step 4:
  ├─> Login successful
  └─> COMPLETE
```

---

## PIPELINE 7: Script Generation

**Назначение**: Генерация Python скриптов для повторяемых действий.

**Entry Point**: 
- `script_generation/real_skyvern_page_ai.py`
- API: `/api/v1/scripts`

### Промпты для генерации

1. **`single-input-action.j2`**
   - Генерирует INPUT_TEXT действие
   - Определяет поле и контекст

2. **`script-generation-input-text-generatiion.j2`**
   - Генерирует код для text input

3. **`script-generation-file-url-generation.j2`**
   - Генерирует код для file upload

4. **`infer-action-type.j2`**
   - Определяет тип действия из контекста

5. **`generate-action-reasoning.j2`**
   - Генерирует комментарии/reasoning

### Output Example

```python
# Generated Script
from skyvern import Skyvern

async def run():
    skyvern = Skyvern()
    
    # Navigate to login page
    await skyvern.goto("https://example.com/login")
    
    # Fill email field
    await skyvern.input_text("#email", "user@example.com")
    
    # Fill password field
    await skyvern.input_text("#password", "********")
    
    # Click login button
    await skyvern.click("#login-btn")
    
    # Extract user data
    data = await skyvern.extract({
        "name": "string",
        "email": "string"
    })
    
    return data
```

---

## Сводная таблица пайплайнов

| Pipeline | Entry Point | Max Iterations | Key Prompts | Output |
|----------|-------------|----------------|-------------|--------|
| **Task V1** | execute_step() | 10-20 steps | extract-action, check-user-goal | Step completion |
| **Task V2** | run_task_v2() | 50 iterations | task_v2, task_v2_generate_* | Workflow |
| **Workflow** | execute_workflow() | N blocks | (block-specific) | Workflow results |
| **Action Extraction** | agent_step() | 1 call/step | extract-action | List[Action] |
| **Data Extraction** | create_extract_action() | 1 call | extract-information | Extracted dict |
| **OTP** | poll_otp_value() | Polling | parse-otp-login | OTP code |
| **Script Gen** | generate_script() | - | script-generation-* | Python code |

---

## Общие паттерны

### 1. Scraping Pattern
```
Все пайплайны используют:
  scrape_website() → ScrapedPage
  └─> elements + screenshots + text
```

### 2. LLM Call Pattern
```
Build prompt → LLM_API_HANDLER() → Parse JSON → Execute
```

### 3. Context Passing
```
Workflow Context (Jinja2)
  ├─> Parameters
  ├─> Block outputs
  └─> Template resolution
```

### 4. Error Handling
```
Try block execution
  ├─> Success → Continue
  ├─> Failure + continue_on_failure=true → Continue
  └─> Failure + continue_on_failure=false → Stop
```

### 5. Browser State
```
BrowserState (persistent)
  ├─> Managed by BrowserManager
  ├─> Reused across steps/blocks
  └─> Closed on completion
```

---

## Рекомендации по выбору пайплайна

**Используйте Task V1 если**:
- Последовательность действий известна
- Простая навигация + extraction
- Нужна скорость выполнения

**Используйте Task V2 если**:
- Сложная задача требует планирования
- Неизвестная последовательность действий
- Нужна адаптивность к изменениям

**Используйте Workflow если**:
- Многошаговый процесс
- Нужна переиспользуемость
- Комбинация разных типов блоков
- Передача параметров между шагами

---

**Документация завершена.** Дата: 2025-11-11

