
Testing Framework

Testing and Evaluation
Once your chatbot works end to end, you need to verify that it handles different messages correctly and produces expected responses. This guide walks you through the full testing workflow: writing test prompts, running the test script against your harness, and evaluating the results using Bedrock Evaluations.

Bedrock Evaluations can't run your harness directly, so instead you invoke the harness on each test prompt, store its responses in a JSONL file, and upload that file to Bedrock Evaluations.

Note: JSONL is a file format where every line represents a separate JSON document. This is in contrast to a JSON file, where the whole file represents a single JSON document.

Automated Testing and Evaluation
This project includes a script that runs your application on a set of prompts. To use it, you need to:

Create a list of test prompts in a separate file
Run the testing script
Evaluate the output using Bedrock Evaluations

1. Write Test Prompts
   Before you can run any automated tests, you need a set of test prompts that cover each route of your chatbot. The goal is to have at least one prompt per category — a bug report, a FAQ question, and an out-of-scope request — so you can verify that your system prompt routes messages to the correct behavior.

Steps
Copy harness-tests-template.json to a new file called harness-tests.json:
cp harness-tests-template.json harness-tests.json
Open harness-tests.json and add the prompts you want to test your application on. The template shows the structure:
{
  "tests": [
    {
      "id": "<unique-test-id></unique>",
      "prompt": "<customer-message></customer>",
      "expected": "<description-of-expected-response></description>"
    }
  ]
}
Screenshot of a harness-tests.json file in the editor with test cases for a bug report, a FAQ question, and an out-of-scope request
An example harness-tests.json covering all three routes

Fill in the three fields for every test case:
Field	Description
id	A unique identifier for the test (e.g. t1_bug_report). Used in log output to identify which test is running.
prompt	The customer message to send to the harness. Write realistic messages that clearly belong to one category.
expected	A description of what a good response should contain. This becomes the reference response for LLM-as-a-judge evaluation. It does not need to be an exact match — it describes the intent so the evaluator can assess whether the actual response is reasonable.
Note: Each test case runs as a single turn in a fresh session (a new runtimeSessionId), so tests cannot influence each other. That also means a test prompt can't rely on earlier conversation turns — for the bug-report route, a good single-turn expected describes the start of the collection behavior (e.g. "acknowledges the bug and asks for the steps to reproduce or the environment"), not the finished ticket.

2. Set Up the Python Environment
   The test script (generate-eval-dataset.py) uses boto3 to call the Bedrock AgentCore API. Before running it, set up a Python virtual environment and install the dependencies (skip this if you already did so during Environment Setup).

Steps
Open a terminal and navigate to project/starter/ in the project repository — the same folder you ran the setup scripts from. Every command on this page runs there.
Create a virtual environment:
python3 -m venv venv
Activate it:
source venv/bin/activate
Install the dependencies:
pip install -r requirements.txt
Verify that boto3 is installed:
python -c "import boto3; print(boto3.__version__)"
This should print 1.43.76 (or newer) without any errors — the Bedrock AgentCore APIs require boto3 1.43+.

Screenshot of a terminal showing pip install -r requirements.txt completing and the boto3 version being printed
Verifying the Python environment

3. Run the Test Script
   The generate-eval-dataset.py script reads your test prompts, invokes the harness once per prompt, and writes the results to a JSONL file. Each line in the output file contains the original prompt, the harness's actual response, and your reference response — everything that Bedrock Evaluations needs to run an LLM-as-a-judge assessment.

There are no IDs to copy out of the console: the script reads the harness and gateway ARNs from agentcore_config.json (written by setup_gateway.py and create_harness.py), attaches the gateway on every invoke so the model can call create_bug_report, and pins the model to us.amazon.nova-pro-v1:0. Run it with:

python generate-eval-dataset.py --tests-json harness-tests.json
If you see an error about a missing harness ARN, run create_harness.py first — it records the ARN in agentcore_config.json.

Screenshot of a terminal showing generate-eval-dataset.py writing one eval line per test case
Running the test script

When the script finishes, check the output file (output_eval_dataset.jsonl). The terminal output lists one wrote eval line message per test case.

Screenshot of output_eval_dataset.jsonl showing one JSON record per line with prompt, referenceResponse, and modelResponses fields
Inspecting the eval dataset

Each line of output_eval_dataset.jsonl is a JSON object with this structure:

{
  "prompt": "Your app crashes every time I try to upload a file...",
  "referenceResponse": "Acknowledges the issue and asks for steps to reproduce...",
  "modelResponses": [
    {
      "response": "I'm sorry to hear about the crash. Could you tell me...",
      "modelIdentifier": "my-support-chatbot"
    }
  ]
}
If any harness call failed, the response field will contain an error message prefixed with [HARNESS_ERROR]. Check the terminal output for details.

4. Create Testing Resources
   Before running evaluations you need an S3 bucket to store the dataset and results, and an IAM role that Bedrock Evaluations can assume. These are defined in cloudformation-testing.yaml.

Steps
Deploy the testing stack:
aws cloudformation deploy   --template-file cloudformation-testing.yaml   --stack-name bug-report-testing-stack   --capabilities CAPABILITY_NAMED_IAM   --region us-east-1
Once the stack is created, retrieve the outputs — you will need the bucket name and role ARN:
aws cloudformation describe-stacks   --stack-name bug-report-testing-stack   --query 'Stacks[0].Outputs'   --output table   --region us-east-1
This prints EvalDatasetBucketName and BedrockEvalRoleArn. Keep these values handy.

5. Run Bedrock Evaluations
   Now that you have a JSONL dataset with your chatbot's responses alongside reference responses, you can use Bedrock Evaluations to assess quality automatically. Bedrock Evaluations supports an LLM-as-a-judge method: an evaluator LLM reads each response and the reference response, and scores how well the chatbot answered.

We use the Bring Your Own Inference (BYOI) approach because our responses come from a file we supply. The JSONL file already contains the harness's responses, so Bedrock Evaluations doesn't need to invoke anything — it only needs to judge the quality.

Upload the Dataset
Upload the JSONL dataset to the S3 bucket created in the previous step:

aws s3 cp output_eval_dataset.jsonl 
  s3://<EvalDatasetBucketName></evaldatasetbucketname>/output_eval_dataset.jsonl 
  --region us-east-1
Note the full S3 URI — you will need it when creating the evaluation job.

Create the Evaluation Job
Use the BedrockEvalRoleArn and EvalDatasetBucketName values from the CloudFormation stack outputs.

aws bedrock create-evaluation-job 
  --job-name support-chatbot-eval-run-1 
  --role-arn <BedrockEvalRoleArn></bedrockevalrolearn> 
  --evaluation-config '{
    "automated": {
      "datasetMetricConfigs": [{
        "taskType": "General",
        "dataset": {
          "name": "support-chatbot-eval-dataset",
          "datasetLocation": {
            "s3Uri": "s3://<EvalDatasetBucketName></evaldatasetbucketname>/output_eval_dataset.jsonl"
          }
        },
        "metricNames": ["Builtin.Correctness"]
      }],
      "evaluatorModelConfig": {
        "bedrockEvaluatorModels": [{
          "modelIdentifier": "amazon.nova-pro-v1:0"
        }]
      }
    }
  }' 
  --inference-config '{
    "models": [{
      "precomputedInferenceSource": {
        "inferenceSourceIdentifier": "my-support-chatbot"
      }
    }]
  }' 
  --output-data-config '{"s3Uri": "s3://<EvalDatasetBucketName></evaldatasetbucketname>/results/"}' 
  --region us-east-1
Replace <BedrockEvalRoleArn></bedrockevalrolearn> and <EvalDatasetBucketName></evaldatasetbucketname> with the values from the CloudFormation stack outputs. The inferenceSourceIdentifier must match the modelIdentifier in your JSONL file — my-support-chatbot is the default written by generate-eval-dataset.py.

To view results in the console, go to Amazon Bedrock(opens in a new tab) → Evaluations in the left sidebar → click on your job once it shows status Completed.

Review the Results
Once the job completes, click on it to see the results.
The results page shows overall scores and per-record breakdowns. The evaluator model scores each response based on how well it matches the intent described in the reference response.
Screenshot of the Bedrock Evaluations results page showing overall correctness scores and per-record breakdowns
Evaluation results page showing scores

Look for patterns in the scores:

Are all three routes producing reasonable responses?
Are any prompts being misrouted (e.g. a bug report getting the "call support" response)?
Are FAQ answers relevant, or is the model missing the point of the question?
Does your application return a correct response, but the LLM-as-a-judge model is marking it as incorrect?
If scores are low for a particular category, iterate on your system prompt: edit system_prompt.txt, re-run create_harness.py, then re-run generate-eval-dataset.py. Common fixes include making the category definitions more specific, tightening the "answer only from the FAQ" instruction, or spelling out the bug-report checklist in more detail.

Next Steps
If you want to expand your test suite, add more test entries to harness-tests.json and re-run the script. Try to improve your application to make sure that it reliably responds to most common use cases.

Cleanup
When you are done with the project, delete the AgentCore resources and the CloudFormation stacks to avoid ongoing charges.

Step 1: Delete the AgentCore Resources
This deletes, in order, the harness, the gateway target, and the gateway — all read from agentcore_config.json:

python cleanup_agentcore.py
Step 2: Empty the S3 Bucket
AWS CloudFormation cannot delete an S3 bucket if there are files inside it, so wipe the evaluation data you uploaded earlier before deleting the testing stack. Skip this and the stack ends up in DELETE_FAILED.

aws s3 rm s3://<EvalDatasetBucketName></evaldatasetbucketname> --recursive --region us-east-1
The bucket name is the EvalDatasetBucketName output of the testing stack — udacity-agentic-engineer-c1-eval-<YOUR_ACCOUNT_ID>. (If the testing stack is already showing DELETE_FAILED, empty the bucket and run its delete-stack command again.)

Step 3: Delete the CloudFormation Stacks
Now we can use the AWS CLI to tear down the infrastructure (Lambda, DynamoDB, IAM roles, and the empty S3 bucket).

Delete the Testing Stack:
aws cloudformation delete-stack --stack-name bug-report-testing-stack --region us-east-1
Delete the Tool Stack:
aws cloudformation delete-stack --stack-name bug-report-tool-stack --region us-east-1

Step 4: Local Cleanup (Optional)
If you want to clean up your local machine or Udacity workspace Python virtual environment to save space, run:

rm -rf venv
