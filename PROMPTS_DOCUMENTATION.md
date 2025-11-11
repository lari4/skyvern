# Документация AI Промптов Skyvern

> **Автор документации**: Автоматически сгенерировано
> **Дата создания**: 2025-11-11
> **Всего промптов**: 55

## Оглавление

1. [Введение](#введение)
2. [Core Task & Action Planning](#1-core-task--action-planning)
3. [Data Extraction](#2-data-extraction)
4. [UI-TARS & CUA Agent](#3-ui-tars--cua-agent)
5. [Validation & Verification](#4-validation--verification)
6. [Form & Input Handling](#5-form--input-handling)
7. [Visual Element Parsing](#6-visual-element-parsing)
8. [Authentication & Verification](#7-authentication--verification)
9. [Task Generation & Workflow](#8-task-generation--workflow)
10. [Script Generation](#9-script-generation)
11. [Error Handling & Debugging](#10-error-handling--debugging)
12. [Schema & Data Structure](#11-schema--data-structure)

---

## Введение

Skyvern использует систему промптов на базе **Jinja2 шаблонов** для управления LLM-агентами, выполняющими автоматизацию веб-браузера. Все промпты находятся в директории `/skyvern/forge/prompts/skyvern/` и загружаются через `PromptEngine`.

### Архитектура промптов:
- **Формат**: Jinja2 templates (`.j2`)
- **Загрузчик**: `PromptEngine` (`skyvern/forge/sdk/prompting.py`)
- **Основные переменные**:
  - `navigation_goal` - цель пользователя
  - `elements` - DOM-элементы страницы
  - `current_url` - текущий URL
  - `navigation_payload_str` - данные пользователя
  - `action_history` - история действий
  - `data_extraction_goal` - цель извлечения данных

### Формат ответов:
Все промпты требуют ответ в **JSON формате** со строго определенной схемой.

---

## 1. Core Task & Action Planning

Эти промпты отвечают за планирование и извлечение действий для навигации по веб-сайтам.

---

### 1.1. `extract-action.j2`

**Назначение**: Основной промпт для извлечения действий из веб-страницы. Анализирует DOM-элементы и скриншот, определяет следующие действия для достижения цели пользователя.

**Используется в**:
- `skyvern/forge/agent.py` (основная логика агента)
- `skyvern/webeye/actions/parse_actions.py`

**Основные действия (action_type)**:
- `CLICK` - клик по элементу
- `INPUT_TEXT` - ввод текста
- `UPLOAD_FILE` - загрузка файла
- `SELECT_OPTION` - выбор опции из выпадающего списка
- `WAIT` - ожидание загрузки
- `SOLVE_CAPTCHA` - решение капчи
- `COMPLETE` - цель достигнута
- `TERMINATE` - завершение с ошибкой

**Шаблон переменных**:
- `{{ navigation_goal }}` - цель пользователя
- `{{ navigation_payload_str }}` - данные пользователя
- `{{ elements }}` - список интерактивных элементов
- `{{ action_history }}` - история предыдущих действий
- `{{ current_url }}` - текущий URL страницы
- `{{ complete_criterion }}` (опционально) - критерий завершения
- `{{ data_extraction_goal }}` (опционально) - цель извлечения данных
- `{{ error_code_mapping_str }}` (опционально) - пользовательские коды ошибок

**Промпт**:
```jinja2
Identify actions to help user progress towards the user goal using the DOM elements given in the list and the screenshot of the website.
Include only the elements that are relevant to the user goal, without altering or imagining new elements.
Accurately interpret and understand the functional significance of SVG elements based on their shapes and context within the webpage.
Use the user details to fill in necessary values. Always satisfy required fields if the field isn't already filled in. Don't return any action for the same field, if this field is already filled in and the value is the same as the one you would have filled in.
MAKE SURE YOU OUTPUT VALID JSON. No text before or after JSON, no trailing commas, no comments (//), no unnecessary quotes, etc.
Each interactable element is tagged with an ID. Avoid taking action on a disabled element when there is an alternative action available.
If you see any information in red in the page screenshot, this means a condition wasn't satisfied. prioritize actions with the red information.
If you see a popup in the page screenshot, prioritize actions on the popup.

Reply in JSON format with the following keys:
{
    "user_goal_stage": str, // A string to describe the reasoning whether user goal has been achieved or not.
    "user_goal_achieved": bool, // True if the user goal has been completed, otherwise False.
    "action_plan": str, // A string that describes the plan of actions you're going to take. Be specific and to the point. Use this as a quick summary of the actions you're going to take, and what order you're going to take them in, and how that moves you towards your overall goal. Output "COMPLETE" action in the "actions" if user_goal_achieved is True. Output "TERMINATE" action in the "actions" if your plan is to terminate the process.
    "actions": array // An array of actions. Here's the format of each action:
    [{
        "reasoning": str, // The reasoning behind the action. This reasoning must be user information agnostic. Mention why you chose the action type, and why you chose the element id. Keep the reasoning short and to the point.
        "user_detail_query": str, // Think of this value as a Jeopardy question and the intention behind the action. Ask the user for the details you need for executing this action. Ask the question even if the details are disclosed in user goal or user details. If it's a text field, ask for the text. If it's a file upload, ask for the file. If it's a dropdown, ask for the relevant information. If you are clicking on something specific, ask about what the intention is behind the click and what to click on. If you're downloading a file and you have multiple options, ask the user which one to download. Examples are: "What product ID should I input into the search bar?", "What file should I upload?", "What is the previous insurance provider of the user?", "Which invoice should I download?", "Does the user have any pets?". If the action doesn't require any user details, describe the intention behind the action.
        "user_detail_answer": str, // The answer to the `user_detail_query`. The source of this answer can be user goal or user details.
        "confidence_float": float, // The confidence of the action. Pick a number between 0.0 and 1.0. 0.0 means no confidence, 1.0 means full confidence
        "action_type": str, // It's a string enum: "CLICK", "INPUT_TEXT", "UPLOAD_FILE", "SELECT_OPTION", "WAIT", "SOLVE_CAPTCHA", "COMPLETE", "TERMINATE"{{', "CLOSE_PAGE"' if has_magic_link_page else ""}}. "CLICK" is an element you'd like to click. "INPUT_TEXT" is an element you'd like to input text into. "UPLOAD_FILE" is an element you'd like to upload a file into. "SELECT_OPTION" is an element you'd like to select an option from. "WAIT" action should be used if there are no actions to take and there is some indication on screen that waiting could yield more actions. "WAIT" should not be used if there are actions to take. "SOLVE_CAPTCHA" should be used if there's a captcha to solve on the screen. "COMPLETE" is used when the {{"complete criterion has been met" if complete_criterion else "user goal has been achieved"}} AND if there's any data extraction goal, you should be able to get data from the page. Never return a COMPLETE action unless the {{ "complete criterion is met" if complete_criterion else "user goal is achieved" }}. "TERMINATE" is used to terminate the whole task with a failure when it doesn't seem like the user goal can be achieved. Do not use "TERMINATE" if waiting could lead the user towards the goal. Only return "TERMINATE" if you are on a page where the user goal cannot be achieved. All other actions are ignored when "TERMINATE" is returned.{{' "CLOSE_PAGE" is used to close the current page when it is impossible to achieve the user goal on the current page.' if has_magic_link_page else ''}}
        "id": str, // The id of the element to take action on. The id has to be one from the elements list
        "text": str, // Text for INPUT_TEXT action only
        "file_url": str, // The url of the file to upload if applicable. This field must be present for UPLOAD_FILE but can also be present for CLICK only if the click is to upload the file. It should be null otherwise.
        "download": bool, // Can only be true for CLICK actions. If true, the browser will trigger a download by clicking the element. If false, the browser will click the element without triggering a download.
        "option": {  // The option to select for SELECT_OPTION action only. null if not SELECT_OPTION action
            "label": str, // the label of the option if any. MAKE SURE YOU USE THIS LABEL TO SELECT THE OPTION. DO NOT PUT ANYTHING OTHER THAN A VALID OPTION LABEL HERE
            "index": int, // the index corresponding to the option index under the select element.
            "value": str // the value of the option. MAKE SURE YOU USE THIS VALUE TO SELECT THE OPTION. DO NOT PUT ANYTHING OTHER THAN A VALID OPTION VALUE HERE
        },
        "click_context": {  // The context for CLICK action only. null if not CLICK action
            "thought": str, // Describe how you decided that this action is a single choice option or multi-choice option.
            "single_option_click": bool, // True if the click is the only choice to proceed towards the goal, regardless of different user context or input. False if there are multiple valid options that depend on user input. Examples: clicking a login button to login is True (it's the only way to login); clicking a radio button for a multi-choice question (e.g., selecting "male", "female", or "other" for gender) is False (the choice depends on user input). When clicking on radio buttons, dropdown options, or any element that represents one of multiple possible selections, this should be False.
        }{% if parse_select_feature_enabled %},
        "context": { // The context for INPUT_TEXT or SELECT_OPTION action only. null if not INPUT_TEXT or SELECT_OPTION action. Extract the following detailed information from the "reasoning", and double-check the information by analysing the HTML elements.
            "thought": str, // A string to describe how you double-check the context information to ensure the accuracy.
            "field": str, // Which field is this action intended to fill out?
            "is_required": bool, // True if this is a required field, otherwise false.
            "is_search_bar": bool, // True if the element to take the action is a search bar, otherwise false.
            "is_location_input": bool, // True if the element is asking user to input where he lives, otherwise false. For example, it is asking for location, or address, or other similar information. Output False if it only requires ZIP code or postal code.
            "is_date_related": bool, // True if the field is related to date input or select, otherwise false.
            "date_format": str, // The format of the date or datetime to be input. For example YYYY-MM-DD, YYYY-MM-DD HH:MM:SS, DD.MM.YYYY, MM/DD/YYYY, etc. If the field is not related to date input or select, this should be null.
        }{% endif %}
    }],{% if verification_code_check %}
    "verification_code_reasoning": str, // Let's think step by step. Describe what you see and think if there is somewhere on the current page where you must enter the verification code now for login or any verification step. Explain why you believe a verification code needs to be entered somewhere or not. Do not imagine any place to enter the code if the code has not been sent yet.
    "place_to_enter_verification_code": bool, // Whether there is a place on the current page to enter the verification code now.
    "should_enter_verification_code": bool, // Whether the user should proceed to enter the verification code.
    "should_verify_by_magic_link": bool // Whether the page instructs the user to check their email for a magic link to verify the login.{% endif %}
}

Consider the action history from the last step and the screenshot together, if actions from the last step don't yield positive impact, try other actions or other action combinations.
Action history from previous steps: (note: even if the action history suggests goal is achieved, check the screenshot and the DOM elements to make sure the goal is achieved)
```
{{ action_history }}
```
{% if complete_criterion %}
Complete criterion:
```
{{ complete_criterion }}
```{% endif %}

User goal:
```
{{ navigation_goal }}
```
{% if error_code_mapping_str %}
Use the error codes and their descriptions to surface user-defined errors. Do not return any error that's not defined by the user. User defined errors:
```
{{ error_code_mapping_str }}
```{% endif %}
{% if data_extraction_goal %}
User Data Extraction Goal:
```
{{ data_extraction_goal }}
```
{% endif %}

User details:
```
{{ navigation_payload_str }}
```

Clickable elements from `{{ current_url }}`:
```
{{ elements }}
```

The URL of the page you're on right now is `{{ current_url }}`.

Current datetime, ISO format:
```
{{ local_datetime }}
```
```

---

### 1.2. `task_v2.j2`

**Назначение**: Промпт для системы Task V2 - планирования многошаговых задач. Генерирует мини-цели (navigate/extract/loop) для движения к общей цели пользователя. Работает как высокоуровневый планировщик, разбивая сложные задачи на управляемые подзадачи.

**Используется в**:
- `skyvern/services/task_v2_service.py:700`

**Типы задач (task_type)**:
- `navigate` - навигация и взаимодействие с элементами (клики, заполнение форм)
- `extract` - извлечение информации из текущей страницы (без навигации)
- `loop` - итерация по списку значений с выполнением одинаковых действий

**Шаблон переменных**:
- `{{ user_goal }}` - общая цель пользователя
- `{{ current_url }}` - текущий URL
- `{{ elements }}` - DOM-элементы страницы
- `{{ task_history }}` - история выполненных задач
- `{{ local_datetime }}` - текущая дата и время

**Промпт**:
```jinja2
You're to assist the user to achieve the user goal in the web, given the DOM elements in the list, the screenshots of the website and the task history list. Plan the next task the user needs to do towards the goal.

You have access to the following task types to take actions:
- navigate: this task can be used to set up a mini goal to achieve in the web which most likely results in navigating the web, like filling a form in the page, clicking a button in the page to open or navigate to another page, interacting with some elements in the page like clicking, typing and selecting options, and so on.
- extract: extract information users would like to output from the page. When planning such a task, you should not expect anything related to navigation mentioned in the "navigate" task. The website will remain still and the only action here is to extract information from it.
- loop: this task can be used to generate a list of planning sessions like this. When to use a loop task? Use loop when there are multiple parallel tasks you can do with the same goal. Each task in the loop has the same goal but with different objects/values/targets/variables. Use loop task when it's in a "breadth first search" situation where you can go through a list of values and execute the same task for each value. Examples:
  - When the goal is "Open up to 10 links from an ecomm search result page, and extract information like the price of each product.", loop task should be used to iterate through a list of links or URLs. In each iteration of the loop, the task will go to the linked page and trigger another planning session with the goal of extracting price information of the product
  - When the goal is "download 5 documents found on a page", loop task should be used to iterate through a list of document names. Each document will trigger another planning session to download the relative document

MAKE SURE YOU OUTPUT VALID JSON. No text before or after JSON, no trailing commas, no comments (//), no unnecessary quotes, etc.

Reply in JSON format with the following keys:
{
  "page_info": str, // Think step by step. Describe all the useful information in the page related to the user goal.
  "extraction_thought": str, // Think step by step. Should any information be extracted given the user goal? If yes, has all the information been extracted? If the user is searching for something, looking for information or specifically trying to extract information along side the goal, consider it an intention to extract information. Phrases like "find something", "show me something", "search something" and so on indicate the intention to extract information.
  "require_extraction": bool, // True if the user goal requires information extraction. False otherwise.
  "task_history_information": str, // Think step by step. In task history, what information has been collected that's helpful and relevant to the user goal, and what information is missing if any.
  "information_extracted": optional[bool], // True if the needed information has been extracted. False if the needed information has not been extracted. If task history has no "extract" type, that means no data extraction has happened, return false. Null if the user goal does not require information extraction.
  "thoughts": str, // Think step by step. What has been done so far and what is the next reasonable mini goal a human can do foreseeably move towards the overall goal.
  "user_goal_achieved": bool, // True if the user goal has been completed, false otherwise. If the user wants to extract information and it has not been done, the user goal is not achieved.
  "plan": str, // The mini goal to achieve to move towards the user goal. DO NOT come up or hallucinate any data that's not provided in the user goal. Be accurate and precise. Return null if the user goal has been achieved.
  "task_type": str, // One of the available task types: navigate, extract, loop
  "loop_values": list[str], // a list of string values to iterate through for loop task. null if it's not a loop task
  "is_loop_value_link": bool, // true if the loop_values is a list of urls to go to before for each planning session inside the loop
}

The URL of the page you're on right now is `{{ current_url }}`.

Clickable elements from the page:
```
{{ elements }}
```

User goal:
```
{{ user_goal }}
```

Task history (the earliest task is the first in the list and the latest is the last in the list):
```
{{ task_history }}
```

Task history explanation:
- completed status means the mini goal has been completed
- terminated and failed mean the mini goal was not fully achieved or couldn't be achieved so you might want to try something else. The reason is given to explain why.

Current datetime, ISO format:
```
{{ local_datetime }}
```
```

---

## 2. Data Extraction

Промпты для извлечения структурированной информации из веб-страниц.

---

### 2.1. `extract-information.j2`

**Назначение**: Извлекает структурированные данные из веб-страницы согласно JSON-схеме. Основной промпт для data extraction - анализирует скриншот и DOM-элементы для получения нужной информации.

**Используется в**:
- `skyvern/webeye/actions/handler.py:3744`
- Генерация скриптов
- Workflow extraction blocks

**Ключевые особенности**:
- Строгое соответствие JSON-схеме выходных данных
- Обработка null-значений для недоступных полей
- Сохранение Jinja-стиля ссылок
- Поддержка пользовательских кодов ошибок

**Шаблон переменных**:
- `{{ data_extraction_goal }}` - цель извлечения данных
- `{{ extracted_information_schema }}` - JSON-схема для выходных данных
- `{{ elements }}` - список элементов страницы
- `{{ current_url }}` - текущий URL
- `{{ extracted_text }}` - текст извлеченный из страницы
- `{{ navigation_payload }}` - данные пользователя
- `{{ previous_extracted_information }}` (опционально) - предыдущий контекст
- `{{ error_code_mapping_str }}` (опционально) - коды ошибок

**Промпт**:
```jinja2
You are given a screenshot, user data extraction goal, the JSON schema for the output data format, and the current URL.

Your task is to extract the requested information from the screenshot and output it in the specified JSON schema format.

Add as much details as possible to the output JSON object while conforming to the output JSON schema.

Do not ever include anything other than the JSON object in your output, and do not ever include any additional fields in the JSON object.

If you are unable to extract the requested information for a specific field in the json schema, please output a null value for that field.

If you are trying to extract the href links which are using the jinja style like "{{}}", please keep the original string.

User Data Extraction Goal: {{ data_extraction_goal }}

Clickable elements from {{ current_url }}:
{{ elements }}

Current URL: {{ current_url }}

Text extracted from the webpage: {{ extracted_text }}

User Navigation Payload: {{ navigation_payload }}

Current datetime, ISO format:
{{ local_datetime }}
```

---

## 3. UI-TARS & CUA Agent

Промпты для GUI-агентов, использующих координатный подход к взаимодействию с UI.

---

### 3.1. `ui-tars-system-prompt.j2`

**Назначение**: Системный промпт для UI-TARS агента - GUI-агент от ByteDance, использующий координатный подход (клики по x,y координатам). Адаптирован из открытого исходного кода UI-TARS.

**Используется в**:
- `skyvern/core/ui_tars_llm_caller.py:33`

**Источник**: Адаптировано из https://github.com/bytedance/UI-TARS (Apache License 2.0)

**Action Space (доступные действия)**:
- `click(point='<point>x1 y1</point>')` - одиночный клик
- `left_double(point='<point>x1 y1</point>')` - двойной клик
- `right_single(point='<point>x1 y1</point>')` - правый клик
- `drag(start_point='<point>x1 y1</point>', end_point='<point>x2 y2</point>')` - перетаскивание
- `hotkey(key='ctrl c')` - горячие клавиши (до 3 клавиш)
- `type(content='xxx')` - ввод текста (с escape-символами)
- `scroll(point='<point>x1 y1</point>', direction='down/up/right/left')` - прокрутка
- `wait()` - ожидание 5 секунд
- `finished(content='xxx')` - завершение задачи

**Шаблон переменных**:
- `{{ language }}` - язык для вывода в секции Thought
- `{{ instruction }}` - инструкция пользователя

**Промпт**:
```jinja2
You are a GUI agent. You are given a task and your action history, with screenshots. You need to perform the next action to complete the task.

## Output Format
Thought: ...
Action: ...

## Action Space

click(point='<point>x1 y1</point>')
left_double(point='<point>x1 y1</point>')
right_single(point='<point>x1 y1</point>')
drag(start_point='<point>x1 y1</point>', end_point='<point>x2 y2</point>')
hotkey(key='ctrl c')
type(content='xxx')
scroll(point='<point>x1 y1</point>', direction='down or up or right or left')
wait()
finished(content='xxx')

## Note
- Use {{language}} in Thought part.
- Write a small plan and finally summarize your next action (with its target element) in one sentence in Thought part.

## User Instruction
{{instruction}}
```

---

## 4. Validation & Verification

Промпты для проверки достижения целей и валидации данных.

---

### 4.1. `check-user-goal.j2`

**Назначение**: Определяет, достигнута ли цель пользователя на текущей странице. Анализирует элементы страницы, скриншот и историю действий для принятия решения о завершении задачи.

**Используется в**:
- `skyvern/forge/agent.py:1845`

**Ключевые особенности**:
- Пошаговое рассуждение (think step by step)
- Поддержка custom complete criterion
- Анализ истории действий
- Обработка новых элементов (emerging elements)

**Шаблон переменных**:
- `{{ navigation_goal }}` - цель пользователя
- `{{ navigation_payload }}` - данные пользователя
- `{{ elements }}` - элементы на странице
- `{{ complete_criterion }}` (опционально) - критерий завершения
- `{{ action_history }}` (опционально) - история действий
- `{{ new_elements_ids }}` (опционально) - ID новых элементов
- `{{ without_screenshots }}` - флаг работы без скриншотов

**Промпт**:
```jinja2
Your are here to help the user determine if the user has completed their goal on the web. Use the content of the elements parsed from the page, the screenshots of the page, the user goal and user details to determine whether the user goal has been completed or not.

Make sure to ONLY return the JSON object in this format with no additional text before or after it:
{
  "page_info": str, // Think step by step. Describe all the useful information in the page related to the user goal.
  "thoughts": str, // Think step by step. What information makes you believe whether user goal has completed or not. Use information you see on the site to explain.
  "user_goal_achieved": bool // True if the user goal has been completed, false otherwise.
}

User Goal:
{{ navigation_goal }}

User Details:
{{ navigation_payload }}

Elements on the page:
{{ elements }}

Current datetime, ISO format:
{{ local_datetime }}
```

---

## 5. Visual Element Parsing

Промпты для интерпретации визуальных элементов (SVG, CSS shapes) в текстовые описания.

---

### 5.1. `svg-convert.j2`

**Назначение**: Конвертирует SVG-элементы в текстовые описания их формы и значения. Помогает LLM понимать визуальные иконки и символы, представленные в SVG-формате.

**Используется в**:
- `skyvern/webeye/scraper/agent_functions.py:223`

**Ключевые особенности**:
- Определение уверенности (confidence) в распознавании
- Флаг распознанности (recognized)
- Краткое описание формы и ее значения

**Шаблон переменных**:
- `{{ svg_element }}` - SVG код элемента

**Промпт**:
```jinja2
You are given a svg element. You need to figure out what its shape means.
SVG Element:
{{svg_element}}

MAKE SURE YOU OUTPUT VALID JSON. No text before or after JSON, no trailing commas, no comments (//), no unnecessary quotes, etc.
Reply in JSON format with the following keys:
{
    "confidence_float": float, // The confidence of the action. Pick a number between 0.0 and 1.0. 0.0 means no confidence, 1.0 means full confidence
    "shape": string, // Concise keywords describing the shape of SVG and its meaning
    "recognized": bool, // False if you can't recognize the shape or you don't understand the meaning, otherwise true.
}
```

---

## 6. Authentication & Verification

Промпты для обработки аутентификации и верификации (OTP, magic links).

---

### 6.1. `parse-otp-login.j2`

**Назначение**: Парсит OTP (One-Time Password) из email или SMS сообщений. Определяет тип OTP (TOTP код или Magic Link) и извлекает значение для автоматической верификации входа.

**Используется в**:
- `skyvern/services/otp_service.py:40`

**Типы OTP**:
- `magic_link` - ссылка для верификации (HTTP/HTTPS URL)
- `totp` - временный код (обычно цифры, иногда с буквами)

**Шаблон переменных**:
- `{{ content }}` - содержимое email/SMS с OTP

**Промпт**:
```jinja2
You receive either an email or a text message containing an OTP(like TOTP, Magic Link) to verify the login. Your job is to parse the content, identify the OTP type and value. There should be only one OTP type and one OTP value in the content. The value must be from the content

You should follow the rules below to identify the OTP type and value:
- If it's a Magic Link login, the value is usually a link which must be a valid HTTP or HTTPS URL.
- If it's a TOTP login, The most common value is a code which is a series of digits, although sometimes it may contain letters.

MAKE SURE YOU OUTPUT VALID JSON. No text before or after JSON, no trailing commas, no comments (//), no unnecessary quotes, etc.

Reply in the following JSON format:
{
    "reasoning": str, // How you figure out what the OTP type and value is or why the OTP type and value is missing. Be precise here to explain the data source and the context that makes you believe where the correct OTP type and value is
    "otp_type": str, // the type of OTP. It can be "magic_link" or "totp"
    "otp_value_found": bool, // true if the OTP value is found. false if the OTP value is not found
    "otp_value": str, // the OTP value. If you cannot identify any OTP value, do not come up with a OTP value and return null
}

Received Content containing OTP:
{{ content }}
```

---

## 7. Workflow Generation

Промпты для генерации workflow из текстовых описаний процедур.

---

### 7.1. `build-workflow-from-pdf.j2`

**Назначение**: Конвертирует Standard Operating Procedures (SOP) из текста в JSON-определение Skyvern workflow. Мощный промпт для автоматического создания рабочих процессов из документации.

**Используется в**:
- `skyvern/services/pdf_import_service.py:172`

**Доступные типы блоков**:
- `login` - аутентификация с credentials
- `navigation` - навигация и заполнение форм
- `action` - клики и простые действия
- `extraction` - извлечение данных
- `task` - сложные задачи (navigation + extraction)
- `file_download` - скачивание файлов
- `for_loop` - циклы по списку
- `validation` - валидация данных
- `wait` - ожидание/пауза
- `code` - выполнение кастомного кода
- `text_prompt` - генерация текста через LLM
- `http_request` - API вызовы

**Типы параметров workflow**:
- `string` - строковые значения
- `json` - JSON объекты
- `credential_id` - пароли/credentials
- `file_url` - URL файлов

**Ключевые требования**:
- Один блок = одно действие
- Сохранение порядка шагов из SOP
- Детальный navigation_goal с точными инструкциями
- engine = "skyvern-1.0" для всех блоков
- Полная конвертация ВСЕХ автоматизируемых шагов

**Шаблон переменных**:
- `{{ sop_text }}` - текст Standard Operating Procedure

**Промпт** (сокращенная версия, см. файл для полной):
```jinja2
You are an AI assistant that converts Standard Operating Procedures (SOP) from text into a Skyvern workflow definition in JSON format.

REQUIRED OUTPUT FORMAT:
{
  "title": "workflow_name",
  "workflow_definition": {
    "parameters": [...],
    "blocks": [...]
  }
}

AVAILABLE BLOCK TYPES:
- "login": For user authentication with credentials
- "navigation": For navigating to pages and filling forms
- "action": For clicking buttons or simple actions
- "extraction": For extracting data from pages
- "task": For complex tasks with both navigation and extraction
- "file_download": For downloading files
- "for_loop": For repeating actions over a list
- "validation": For validating extracted data
- "wait": For waiting/pausing
- "code": For custom code execution
- "text_prompt": For LLM text generation
- "http_request": For API calls

CRITICAL INSTRUCTIONS:
1. BE THOROUGH: Convert EVERY automatable step from the SOP into a block
2. PRESERVE SPECIFICITY: Keep exact instructions from SOP
3. ONE ACTION PER BLOCK: Each block does ONE specific thing
4. DETAILED navigation_goal: Copy exact instructions from SOP
5. MAINTAIN ORDER: Keep exact order of steps from SOP
6. INCLUDE ALL CONDITIONS: Create conditional blocks if needed
7. Set engine to "skyvern-1.0" for all blocks that need it

Standard Operating Procedure:
{{ sop_text }}

Return ONLY a valid JSON object.
```

---

## 8. Task Generation & Schema Creation

Промпты для автоматической генерации задач и JSON-схем из пользовательских запросов.

---

### 8.1. `generate-task.j2`

**Назначение**: Генерирует определение задачи (task) из текстового промпта пользователя. Автоматически определяет URL, цели навигации и извлечения данных, создавая структурированную задачу для агента.

**Используется в**:
- `skyvern/services/task_v1_service.py:48`

**Поля task schema**:
- `url` (required) - стартовый URL (только HTTPS)
- `suggested_title` (required) - короткое название задачи
- `navigation_goal_reasoning` (required) - объяснение необходимости navigation
- `is_navigation_goal_required` (required) - нужна ли навигация
- `navigation_goal` (optional) - цель навигации с критериями завершения ("COMPLETE when...")
- `data_extraction_reasoning` (required) - объяснение необходимости extraction
- `require_extraction` (required) - нужно ли извлечение данных
- `data_extraction_goal` (optional) - цель извлечения данных
- `navigation_payload` (optional) - JSON с данными для заполнения форм

**Ключевые требования**:
- Минимум одна цель (navigation_goal или data_extraction_goal) обязательна
- Фразы типа "find", "show me", "search" указывают на extraction
- navigation_goal должен включать explicit completion criteria

**Шаблон переменных**:
- `{{ user_prompt }}` - запрос пользователя

**Промпт** (сокращенная версия):
```jinja2
We are building an AI agent that can automate browser tasks. The task creation schema is a JSON object with the following fields:

url: str. Required. Starting URL for the task (HTTPS only).
suggested_title: str. Required. Short title describing the task.
navigation_goal_reasoning: str. Required. Why navigation goal is needed.
is_navigation_goal_required: bool. Required. Whether navigation goal is required.
navigation_goal: str. Optional. Actions needed to achieve task. Must include completion criteria: "COMPLETE when...".
data_extraction_reasoning: str. Required. Think step by step. Should any information be extracted?
require_extraction: bool. Required. Is data extraction required?
data_extraction_goal: str. Optional. Goal in terms of extracting data.
navigation_payload: json. Optional. Information to complete task (form values, parameters).

At least one of navigation_goal or data_extraction_goal should be provided.

Respond with only JSON output for:
{{ user_prompt }}
```

---

### 8.2. `suggest-data-schema.j2`

**Назначение**: Генерирует JSON-схему для извлечения данных на основе пользовательского запроса. Автоматически создает структуру данных, которую должен вернуть агент при extraction.

**Используется в**:
- `skyvern/forge/agent_protocol.py:2457`

**Ключевые особенности**:
- Использует дополнительный контекст (data_extraction_goal, existing_schema)
- Генерирует typed JSON schema с descriptions
- Поддерживает вложенные объекты и массивы
- Включает required fields

**Шаблон переменных**:
- `{{ input }}` - запрос пользователя
- `{{ additional_context }}` (optional) - дополнительный контекст

**Пример**:
```
Input: "Generate a data schema that extracts the title and link for the posts as a list"
Context: {"url": "https://news.ycombinator.com", "data_extraction_goal": "Extract the title and link of the top 5 posts"}
Output: {"posts": {"type": "array", "items": {"type": "object", "properties": {...}}}}
```

**Промпт**:
```jinja2
We are developing an interface for AI agent tasks that use JSON schemas to describe the shape of the data to that needs to extracted from a web page.

You are given an input prompt from a user, and some additional context. Your goal is to generate a JSON schema given the user prompt and the context.

If additional context is given, try to use it for further clues about the data that needs to be extracted.

Example:
User prompt: Generate a data schema that extracts the title and link for the posts as a list
Additional context: 
{"url": "https://news.ycombinator.com", "data_extraction_goal": "Extract the title and link of the top 5 posts"}

Suggested Data Schema: 
{
  "posts": {
    "type": "array",
    "items": {
      "type": "object",
      "properties": {
        "title": {"type": "string", "description": "Title of the post"},
        "link": {"type": "string", "description": "Link to the post"}
      },
      "required": ["title", "link"]
    }
  }
}

Additional context: {{additional_context}}

Respond only with JSON output containing a single key "output" with the value of the suggested data schema given:
{{ input }}
```

---

## 9. Form & Input Handling

Промпты для точного взаимодействия с формами и input-элементами.

---

### 9.1. `single-input-action.j2`

**Назначение**: Специализированный промпт для выполнения одного INPUT_TEXT действия. Более фокусированный подход к заполнению текстовых полей по сравнению с общим extract-action.

**Используется в**:
- `skyvern/core/script_generations/real_skyvern_page_ai.py:291`

**Ключевые особенности**:
- Только одно действие INPUT_TEXT
- Детальный контекст для поля (required, search_bar, location_input, date_related)
- Определение формата даты (date_format)
- Проверка verification code

**Возвращаемый контекст**:
- `field` - название поля
- `is_required` - обязательное поле или нет
- `is_search_bar` - является ли поисковой строкой
- `is_location_input` - запрашивает ли адрес/локацию
- `is_date_related` - связано ли с датой
- `date_format` - формат даты (YYYY-MM-DD, DD.MM.YYYY и т.д.)

**Шаблон переменных**:
- `{{ navigation_goal }}` - инструкция пользователя
- `{{ navigation_payload_str }}` - данные пользователя
- `{{ elements }}` - HTML элементы
- `{{ current_url }}` - текущий URL
- `{{ verification_code_check }}` - флаг проверки verification code

**Промпт** (сокращенная версия):
```jinja2
Your are here to help the user to perform an INPUT_TEXT action on the web page.
Each actionable element is tagged with an ID. Only take the action on the elements provided.
MAKE SURE YOU OUTPUT VALID JSON.

Reply in JSON format:
{
    "page_info": str,
    "thoughts": str,
    "actions": [{
        "reasoning": str,
        "user_detail_query": str,
        "user_detail_answer": str,
        "confidence_float": float,
        "action_type": "INPUT_TEXT",
        "id": str,
        "text": str,
        "context": {
            "thought": str,
            "field": str,
            "is_required": bool,
            "is_search_bar": bool,
            "is_location_input": bool,
            "is_date_related": bool,
            "date_format": str
        }
    }]
}

User instruction: {{ navigation_goal }}
User details: {{ navigation_payload_str }}
HTML elements from {{ current_url }}: {{ elements }}
```

---

### 9.2. `extraction_prompt_for_nat_language_loops.j2`

**Назначение**: Извлекает значения для loop-итераций из естественного языка и содержимого страницы. Определяет, по каким элементам нужно итерировать (URLs, names, titles и т.д.).

**Используется в**:
- `skyvern/workflow/models/block.py:1233`

**Примеры использования**:
- "go to each product page" → извлекает URLs продуктов
- "extract the authors of the top 4 posts" → извлекает имена авторов
- "summarize the text of each article" → извлекает URLs статей
- "download each file" → извлекает URLs файлов
- "check if these articles are AI-related" → извлекает заголовки статей

**Возвращает**: `loop_values` - массив строк для итерации

**Шаблон переменных**:
- `{{ natural_language_prompt }}` - запрос пользователя

**Промпт**:
```jinja2
Analyze the current webpage and extract information based on this request: {{ natural_language_prompt }}

You need to identify what values should be iterated over (loop_values). Each value should be the primary data that will be used in each loop iteration.

For example:
- If the request is "go to each product page", extract product URLs as strings
- If the request is "extract the authors of the top 4 posts", extract author names as strings
- If the request is "summarize the text of each article", extract article URLs as strings
- If the request is "download each file", extract file URLs as strings
- If the request is "check if these articles are AI-related", extract article titles as strings

Return the results in the specified schema format with loop_values containing an array of strings.
```

---

