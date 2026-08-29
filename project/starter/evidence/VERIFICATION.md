# Verification Evidence

Verification date: 2026-08-23
AWS region: `us-east-1`

## Direct Lambda and DynamoDB verification

The direct Lambda test returned HTTP status 200 with:

```json
{"ticketId": "f15261cc-c426-4a81-85f8-5310e01b537d", "status": "OPEN"}
```

The corresponding DynamoDB item contains the submitted description, reproduction steps, environment, `OPEN` status, and creation timestamp.

## Successful multi-turn bug transcript

Session: `446afd1b-83d4-4bb4-8356-663205bcbdd6-support-chat`

```text
Customer: The cart page turns completely blank when I select checkout.
Assistant: I'm sorry about that. Can you please describe the steps you take before the cart page turns blank?

Customer: I add a blue shirt to the cart, open the cart, and select Checkout.
Assistant: Thank you for the details. Could you please tell me which browser, operating system, and device you are using?

Customer: Firefox 128 on an Ubuntu 24.04 desktop.
[tool call] bugreports___create_bug_report
Assistant: Thank you for reporting this issue. We have created a bug report with the ticket ID 5757db94-a1e4-4afb-8408-702691aa0fd0. Our team will look into it.
```

## FAQ route

```text
Customer: How can I track my order?
Assistant: Once your order ships, we'll email a tracking link. If you have an account, you can also find tracking under My Orders.
```

## Human-handoff route

```text
Customer: Can you write a birthday poem for my friend?
Assistant: This is a request that requires human creativity and is outside the scope of my capabilities. Please call our human support at 1-800-555-0199 (Monday-Friday).
```

## Automated evaluation

- Job: `support-chatbot-eval-run-1`
- Job ID: `1xckdqssmm2v`
- Status: `Completed`
- Metric: `Builtin.Correctness`
- Records: 7
- Score: `1.0` for every record; overall mean `1.0`
- Local result artifact: `evidence/evaluation-results/`

The seven cases cover incomplete bug collection, informal reproduction steps, FAQ grounding, out-of-scope handoff, ambiguous bug priority, and prompt injection.
