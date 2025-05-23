# Agent Design Part D: Communication & Permissions

This document details Agent-to-Agent communication mechanisms and the agent permissions management framework. Refer to `01a_agent_overview_and_soa.md` for the general agent overview.

## 4. Agent-to-Agent Communication / Workflow

*   **Primary Mechanism:** Asynchronous task/message dispatching via the `SystemOrchestratorAgent` (SOA) or `AgentManager`. These modules would typically use an internal message queue or a similar robust mechanism for reliable task routing.
    *   Example: An Analyst Agent generates a trading signal. It then calls the `dispatch_task_to_agent` tool. The `SystemOrchestratorAgent` (if it's coordinating) or the `AgentManager` (if it's a direct pre-configured link) receives this tool call and routes the signal (as a structured task payload) to the appropriate Trader Agent instance.
*   **Shared Context (Limited & Controlled):** While direct messaging or task dispatching is primary for actionable signals, some high-level, slowly changing contextual information (e.g., a global market sentiment score generated by a dedicated Analyst Agent, or a summary of active high-impact news) could be written by an authorized agent to a well-defined shared memory location (e.g., a specific YAML file in a common area of the agent memory cache, managed by `MemoryBankManager`). Other agents could then be prompted to read this shared context periodically. This approach should be used sparingly to maintain loose coupling between agents.
*   **SOA as Central Hub for User-Driven Interactions:** For complex operations initiated by the user via natural language, the `SystemOrchestratorAgent` often acts as the central coordinator. It receives the user's high-level request, breaks it down (potentially through multiple LLM turns and tool calls), and then dispatches specific sub-tasks or configuration commands to the relevant Analyst or Trader agents via the `AgentManager` or `dispatch_task_to_agent` tools.

## 5. Agent Permissions Management

*   **5.1 Storage & Configuration:**
    *   Permissions for each agent instance will be defined in its configuration, likely stored as a dedicated section within its main `agent_config.yaml` file located in its specific memory directory (e.g., `{cache_dir}/agent_memory/{agent_instance_id}/agent_config.yaml`).
    *   These configurations are initially set when an agent is created (e.g., by the `SystemOrchestratorAgent` based on user input or a template) and managed by the `AgentManager`. Modifications to permissions would also go through the SOA/AgentManager.

*   **5.2 Enforcement Point:**
    *   The `ToolExecutor` module will serve as the primary enforcement point for critical and sensitive actions, especially those involving financial transactions or significant state changes.
    *   Before executing a tool call like `create_order`, `cancel_order`, `set_position_tp_sl`, `write_agent_memory_file` (for certain critical files), or `send_notification` (to prevent spam), the `ToolExecutor` will:
        1.  Identify the `agent_instance_id` that initiated the tool call.
        2.  Instruct the `AgentManager` (or a dedicated permission service) to retrieve and evaluate the permissions for that specific agent instance against the requested tool and its parameters.
        3.  Only if the action is permitted will the `ToolExecutor` proceed with the actual tool execution. Otherwise, an error is returned to the calling LLM.

*   **5.3 Permission Granularity (Examples):**
    *   `can_trade_symbols`: List of allowed trading symbols (e.g., `["BTCUSDT", "ETHUSDT"]`) or a wildcard (e.g., `"ALL_BINANCE_FUTURES"`). Can be `null` or an empty list for non-trading agents.
    *   `max_position_size_usd_per_symbol`: A map defining max USD value per symbol (e.g., `{"BTCUSDT": 10000, "ETHUSDT": 5000}`).
    *   `max_total_open_positions_value_usd`: Overall limit for the agent's portfolio.
    *   `max_order_quantity_per_trade`: Symbol-specific limits (e.g., `{"BTCUSDT": 0.5, "ETHUSDT": 10}`).
    *   `allowed_order_types`: List of order types the agent can place (e.g., `["MARKET", "LIMIT", "STOP_MARKET"]`).
    *   `can_schedule_background_jobs`: Boolean (typically true for SOA or specialized Analyst Agents).
    *   `can_send_notifications_channels`: List of allowed notification channels (e.g., `["TELEGRAM", "UI_ALERT"]`).
    *   `can_create_new_agents`: Boolean (typically only true for SOA).
    *   `can_modify_own_strategy_autonomously_level`: Enum (e.g., `NONE`, `PARAMETERS_ONLY`, `LIMITED_LOGIC_ADAPTATION`) - to control self-learning capabilities.

*   **5.4 Default Permission Profiles:**
    *   The system can provide predefined permission templates (e.g., "ReadOnly_MarketAnalyst", "Cautious_TrendTrader", "Aggressive_Scalper_LowCap", "SystemOrchestrator_FullAccess").
    *   When the `SystemOrchestratorAgent` initiates the creation of a new agent instance via `create_new_agent_instance` tool, it can specify one of these profiles, which the `AgentManager` will then use to populate the initial `agent_config.yaml` for the new agent. Users could potentially fine-tune these later via SOA.