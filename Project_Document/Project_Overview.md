Customer Support Chatbot with Amazon Bedrock AgentCore
In this project you will build a customer support chatbot using the Amazon Bedrock AgentCore managed harness. The chatbot handles customers' messages for a fictional online shop and must route each message to the correct behavior based on its type:

Bug reports — collect the details over the conversation, then file a ticket in a database using the create_bug_report tool
Platform questions — answer common questions about orders, shipping, returns, and payments using an embedded FAQ
Other requests — politely redirect the customer to a human support phone line
Why AgentCore? Bedrock Agents Classic was closed to new customers on July 30, 2026, so this course uses its successor, the AgentCore managed harness. Bedrock Evaluations — which you'll use for testing — is unaffected.

How the Chatbot Should Behave
The centerpiece of this project is prompt engineering: all of the routing, information gathering, and grounding behavior lives in a single system prompt that you design. The harness supplies the agent loop — model calls, session memory, and tool execution — and your prompt supplies the behavior. There are no condition nodes or separate classifiers:

If the customer is reporting a bug, the chatbot collects the bug description, the steps to reproduce, and the customer's environment through conversation. Because harness sessions are stateful across turns, it can simply ask for whatever is missing. Once everything is collected, it files a ticket with the create_bug_report tool — a Lambda function exposed to the model through an AgentCore Gateway — and tells the customer their ticket ID.
If the customer has a question about the platform (orders, shipping, returns, payments), the chatbot answers using only the FAQ document embedded in the prompt.
If the request doesn't fit either category, the chatbot politely redirects the customer to a human support line.
Your job is to write the system prompt that classifies each incoming message and produces the right behavior. How you describe the categories, structure the bug-report checklist, and phrase the routing rules is up to you.


Available Resources
The project starter includes:

system_prompt.txt — your main deliverable: the system prompt for the chatbot
cloudformation-tool.yaml — deploys the DynamoDB ticket table, the create_bug_report Lambda, and the IAM roles for the gateway and the harness
create_bug_report.py — the Lambda function code (also embedded in the tool template) that stores bug reports in DynamoDB
setup_gateway.py — creates the AgentCore Gateway and registers the Lambda as the create_bug_report tool
create_harness.py — creates (or updates) the managed harness from system_prompt.txt
chat.py — a terminal chat client for trying out your chatbot in a multi-turn session
online_shop_faq.md — a fictional FAQ document for answering platform questions
harness-tests-template.json — template for developing your test suite
generate-eval-dataset.py — runs your harness against your test suite and produces a JSONL dataset for Bedrock Evaluations
cloudformation-testing.yaml — deploys the S3 bucket and IAM role needed for evaluation
cleanup_agentcore.py — deletes the harness, gateway target, and gateway when you're done
requirements.txt — Python dependencies

Technology Stack
Amazon Bedrock AgentCore managed harness(opens in a new tab) — runs the chatbot: agent loop, stateful sessions, and tool execution
Amazon Bedrock AgentCore Gateway(opens in a new tab) — exposes the bug report Lambda as a tool
Amazon Bedrock Evaluations(opens in a new tab) — LLM-as-a-judge evaluation
AWS Lambda(opens in a new tab) — bug report tool runtime
Amazon DynamoDB(opens in a new tab) — bug report storage

What You Will Demonstrate
Prompt engineering for reliable message routing inside a single system prompt
Multi-turn information gathering using stateful harness sessions
Tool use through an AgentCore Gateway (Lambda + DynamoDB)
Embedding reference documents in prompts for FAQ answering
Automated testing with Bedrock Evaluations using LLM-as-a-judge
