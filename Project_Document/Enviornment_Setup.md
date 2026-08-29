
Getting Started
Dependencies
Before starting the project, make sure you have the following:

An AWS account with Amazon Bedrock and Amazon Bedrock AgentCore access enabled
AWS CLI configured with appropriate credentials
Python 3.9+ with boto3 1.43+ installed (pip install -r requirements.txt)
Access to the Amazon Nova Pro model. This project pins us.amazon.nova-pro-v1:0 everywhere — do not rely on the harness default model, which requires an AWS Marketplace subscription that lab accounts cannot complete.
Region: Use us-east-1 for all resources in this project. Some smaller regions may not have all Bedrock AgentCore features available; the scripts and templates assume us-east-1.

Note: Bedrock Agents Classic was closed to new customers on July 30, 2026. This project runs on its successor, the Amazon Bedrock AgentCore managed harness, with the bug-report tool exposed through an AgentCore Gateway. Bedrock Evaluations, which you will use for testing, is unaffected.

Make sure your AWS region is set to us-east-1 — every command, script, and template in this project assumes it.

Project Files
All of the files below live in project/starter/ in the project repository. Run every command in this project from that folder.

File	Description
cloudformation-tool.yaml	CloudFormation template that creates the DynamoDB table, the create_bug_report Lambda, and the IAM roles for the gateway and the harness
cloudformation-testing.yaml	CloudFormation template that creates the evaluation resources (S3 bucket + evaluation role)
create_bug_report.py	The Lambda function code (also embedded in the tool template) that stores bug reports in DynamoDB
setup_gateway.py	Creates the AgentCore Gateway and registers the Lambda as the create_bug_report tool
system_prompt.txt	Your main deliverable — the system prompt for the chatbot
create_harness.py	Creates (or updates) the managed harness from system_prompt.txt; re-run it every time you change your prompt
chat.py	A terminal chat client for trying out your chatbot in a multi-turn session
online_shop_faq.md	Fictional FAQ document covering orders, shipping, returns, payments, products, account, and privacy
harness-tests-template.json	Template for developing your test suite
generate-eval-dataset.py	Runs your harness against a test suite and produces a JSONL file for Bedrock Evaluations
cleanup_agentcore.py	Deletes the harness, gateway target, and gateway when you're done
requirements.txt	Python dependencies (boto3 1.43+)
Step 1: Deploy the Tool Stack and Create the Gateway
When a customer reports a bug, the chatbot needs to persist it somewhere so the engineering team can follow up. This project uses a DynamoDB table as a simple ticket store and a Lambda function as the tool implementation. The chatbot reaches the Lambda through an AgentCore Gateway — the gateway presents the Lambda to the model as a callable tool named create_bug_report.

1. Deploy the tool stack (DynamoDB table + Lambda + IAM roles):

aws cloudformation deploy   --template-file cloudformation-tool.yaml   --stack-name bug-report-tool-stack   --capabilities CAPABILITY_NAMED_IAM   --region us-east-1
The --capabilities CAPABILITY_NAMED_IAM flag is required because the template creates named IAM roles. Besides the Lambda and the table, the stack creates two AgentCore roles whose ARNs appear in the stack outputs: a gateway role (lets the gateway invoke the Lambda) and a harness execution role (lets the harness call Bedrock models and invoke the gateway).

2. Create the gateway and register the tool:

python setup_gateway.py
The script reads the stack outputs itself — no copy-pasting required — and saves everything the later steps need to agentcore_config.json.

If setup_gateway.py fails right after the stack finishes with an access or validation error mentioning the role, that's IAM propagation delay — the script already retries, but if it still fails just run it again a minute later.

You will create the harness itself with create_harness.py after writing your system prompt — see the Instructions page.
