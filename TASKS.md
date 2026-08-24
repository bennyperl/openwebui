# Local Setup & Task Tracking

This document logs local environment setup steps, custom fixes, and completed tasks for this Open WebUI repository instance.

---

## Task Log

### Task 1: Local Environment Setup & Windows Fixes
* **Date:** August 24, 2026
* **Status:** Completed
* **Commit:** `6666346f5` (`fix(local-setup): fix Windows secret key generation and set ThreadedResolver for Windows local setup`)
* **Summary of Changes:**
  1. **Fixed `start_windows.bat` Secret Key Generation:**
     * Resolved batch script syntax errors (`SET /P` redirection failure) that generated repeated `"The system cannot find the file specified"` errors and prevented `.webui_secret_key` creation.
     * Implemented secure random key generation using Python's standard `secrets` module.
  2. **Fixed OpenRouter / External API DNS Resolution (`ClientConnectorDNSError`):**
     * Fixed an issue where `aiohttp` using `aiodns` / `c-ares` failed DNS lookups for `openrouter.ai` on Windows hosts.
     * Configured `aiohttp.ThreadedResolver()` in `backend/open_webui/utils/session_pool.py` and `backend/open_webui/routers/openai.py` to use native OS system DNS resolution (`getaddrinfo`).
  3. **Configured OpenRouter API Integration:**
     * Setup API Base URL (`https://openrouter.ai/api/v1`) and API key configuration for external LLM routing.

### Task 2: Make User Interface Settings Save via Manual Save Button
* **Date:** August 24, 2026
* **Status:** Completed
* **Summary of Changes:**
  * Updated User Interface settings (`src/lib/components/chat/Settings/Interface.svelte`) so that settings are no longer auto-saved on every single toggle/input change.
  * Aligned User Interface settings behavior with Admin Interface settings: all option changes bind locally, and backend/store updates are triggered only when clicking the form's **Save** button.

### Task 3: File System Prompt Injection Feature
* **Date:** August 24, 2026
* **Status:** Completed
* **Summary of Changes & Architectural Decision:**
  * **UX & Interaction Design:** Added an interactive System Prompt trigger chip (`+ Add File System Prompt`) and an expandable inline editing container directly above the message composer (`src/lib/components/chat/MessageInput.svelte`).
  * **Design Rationale & Decisions:**
    1. **Single System Prompt for File Attachments:** For now, one unified system prompt controls instructions across all attached files in the prompt. This avoids clutter while providing clear guidance to the model.
    2. **Quick Actions for Convenience:** Integrated quick template chips (`Summarize`, `Extract Data`) to allow users to quickly insert common file instructions with one click.
    3. **On-Demand (Click-to-Open) Container:** Since file-specific system prompt overrides can be either a day-to-day need or an edge case depending on the user, the container opens **only on click** and is not expanded by default, keeping the chat input clean.
    4. **Applied Indicator & Save Feedback:** Added an **"Apply & Save"** button inside the container that triggers a toast notification (`"File system prompt saved and applied!"`) and turns the chip into an active state indicator (`⚡ System Prompt Active` with a pulsing green status dot) so the user has clear, immediate feedback that their system prompt is applied.
    5. **Post-Submission Reset:** Upon sending the message in `submitHandler`, `params.system` is automatically cleared so the custom file system prompt does not remain active for subsequent messages in the chat session.
  * **State Binding:** Connected `params.system` binding through `Chat.svelte`, `Placeholder.svelte`, and `MessageInput.svelte` so the system prompt flows to the model request payload and persists with the chat session.

---

## Future Tasks / Backlog

### Task 4 (Planned): File System Prompt Enhancements & Bug Fixes
* **Status:** Planned / Backlog
* **Objective:** Further improve usability, bug fixes, and visibility for custom file system prompts:
  * **Bug - Edit Message with Files missing Custom System Prompt:** Fix an issue where editing an existing message that contains attached files does not display or allow customizing the system prompt option (`FileSystemPrompt` component missing in message edit context).
  * **Recent System Prompts History Quick Pick:** Display the 2 most recently used custom system prompts (retrieved from the current/previous chat history) as quick template buttons inside the `FileSystemPrompt` drawer so users can re-use their latest custom prompts with one click.
  * **Tooltip for Custom System Prompt:** Add an interactive tooltip to the `⚡ System Prompt Active` chip showing a full preview of the custom system prompt when hovered/focused.
  * **Chat History Indicator for File System Prompt:** Add a visual badge/indicator in past chat messages (e.g. on file attachment components or message header) indicating that the file/query was sent with a custom system prompt, with the ability to click or hover to view the exact injected system prompt text.

### Task 5 (Planned): Create Generic Helpers for Settings Management
* **Status:** Planned / Backlog
* **Objective:** Refactor settings components (e.g. `src/lib/components/chat/Settings/Interface.svelte`) by creating generic handler utilities instead of duplicating logic across individual fields:
  * **Generic Boolean Toggle Helper:** A reusable function/handler to toggle boolean fields without repeating individual `toggle...()` functions for each switch.
  * **Generic Field Update Helper:** A standardized input change handler to update target setting keys/fields dynamically.
