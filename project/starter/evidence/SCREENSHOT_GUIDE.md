# Required Screenshot Guide

The implementation and evaluation are complete. Capture these screenshots before submitting because authenticated AWS Console screenshots cannot be generated from the local CLI workspace.

## 1. System prompt

Open `system_prompt.txt` in the editor and capture the route definitions plus the bug-report collection/tool rules. A second screenshot can show the `{{FAQ}}` placeholder and final enforcement rules if the complete prompt does not fit on one screen.

## 2. Multi-turn bug conversation

From Command Prompt:

```cmd
conda activate udacity-p1
set AWS_PROFILE=udacity-agentcore
cd C:\Users\Iflal\Desktop\PRAC\UDACITY\Project_1_Customer_Support_ChatBot_AWS_BedRock_AgentCore\project\starter
python chat.py
```

Use these messages one at a time:

```text
The cart page turns completely blank when I select checkout.
I add a blue shirt to the cart, open the cart, and select Checkout.
Firefox 128 on an Ubuntu 24.04 desktop.
```

Capture the follow-up questions, `[tool call] bugreports___create_bug_report`, and returned ticket ID. This creates one additional valid test ticket.

## 3. DynamoDB ticket

In AWS Console, select `us-east-1`, then open:

`DynamoDB` → `Tables` → `bug-report-tool-stack-bug-reports` → `Explore table items`

Capture a complete `OPEN` ticket showing `ticketId`, `description`, `stepsToReproduce`, and `environment`. Ticket `5757db94-a1e4-4afb-8408-702691aa0fd0` is the already-verified chatbot ticket.

## 4. FAQ and handoff routes

Start a fresh `python chat.py` run for each prompt and capture the responses:

```text
How can I track my order?
```

```text
Can you write a birthday poem for my friend?
```

The first must give the FAQ tracking answer. The second must provide `1-800-555-0199 (Monday-Friday)` without writing the poem.

## 5. Bedrock Evaluation result

In AWS Console, select `us-east-1`, then open:

`Amazon Bedrock` → `Evaluations` → `support-chatbot-eval-run-1`

Capture the `Completed` status and correctness score. All seven records scored `1.0`.

## Submission files

Include at minimum:

- `system_prompt.txt`
- `harness-tests.json`
- `output_eval_dataset.jsonl`
- `SUBMISSION_NOTES.md`
- Required screenshots

Do not submit `.aws` credential files or copy access keys/session tokens into any project file.
