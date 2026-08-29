# Customer Support Chatbot — Submission Notes

## Implementation summary

The chatbot uses one system prompt to route each message to exactly one behavior:

1. Bug reports: collect the description, reproduction steps, and environment across the stateful conversation; call the bug-report tool only when all fields are available; return the resulting ticket ID.
2. Platform questions: answer only from the embedded FAQ.
3. Other or unsupported requests: politely provide the FAQ's human-support phone number.

The prompt also defines precedence for ambiguous bug messages and rejects prompt-injection attempts.

## Evidence checklist

- [x] Final `system_prompt.txt` with the unchanged `{{FAQ}}` placeholder
- [x] Successful Lambda test response showing `ticketId` and `OPEN` status
- [x] DynamoDB item showing the same ticket and all three bug fields
- [x] Multi-turn terminal chat showing collection of all missing fields
- [x] Successful `bugreports___create_bug_report` tool call and ticket ID response
- [x] Covered FAQ question and grounded response
- [x] Uncovered or unrelated request and phone handoff
- [x] `harness-tests.json` covering all three routes
- [x] `output_eval_dataset.jsonl` without `[HARNESS_ERROR]` responses
- [x] Screenshot of the completed Bedrock Evaluation results page
- [x] Written evaluation observations below

## Evaluation observations

- Evaluation job name: `support-chatbot-eval-run-1`
- Evaluation date: 2026-08-23
- Overall correctness score: `1.0` (7 of 7 records scored `1.0`)
- Bug-report route observations: The chatbot asks for exactly one missing field at a time, recognizes informal reproduction sequences, prioritizes concrete malfunctions over general questions, and waits for all three customer-provided fields before successfully creating a ticket.
- FAQ route observations: Order-tracking and payment-method answers are concise and match only the embedded FAQ.
- Human-handoff route observations: Unrelated and prompt-injection requests are refused and redirected to `1-800-555-0199` without completing the embedded task.
- Changes made after reviewing low-scoring cases: Pre-evaluation manual testing exposed premature tool calls and cross-session memory retrieval. The prompt gained a post-FAQ tool gate and explicit examples; the clients now pass a unique `actorId` for memory isolation; placeholder values are rejected by Lambda; private `<thinking>` blocks are removed from customer-visible output. Automated pre-evaluation responses were refined until all expected behaviors matched.
- Final conclusion: All three routes and the prompt-injection edge case behaved correctly, all seven harness invocations succeeded, and Bedrock Evaluations assigned perfect correctness to every record.

## Architecture

This submission follows the current Amazon Bedrock AgentCore managed-harness rubric and uses an AgentCore Gateway to expose the Lambda tool.
