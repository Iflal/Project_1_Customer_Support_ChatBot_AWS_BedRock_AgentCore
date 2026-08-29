
## Project Instructions

### Step 1: Create Resources for Your Application

You already deployed the tool stack and created the gateway on the **Environment Setup** page. As there, every command on this page runs from `project/starter/` in the project repository. Here is a recap of what `cloudformation-tool.yaml` created, and how to verify the tool works in isolation before you build your prompt around it.

| Resource        | Name                                        | Purpose                                                                      |
| --------------- | ------------------------------------------- | ---------------------------------------------------------------------------- |
| DynamoDB table  | `bug-report-tool-stack-bug-reports`       | Stores one item per bug report, keyed by`ticketId`                         |
| Lambda function | `bug-report-tool-stack-create-bug-report` | The`create_bug_report` tool implementation — writes tickets to DynamoDB   |
| IAM role        | `bug-report-tool-stack-lambda-role`       | Grants the Lambda permission to write logs and call`PutItem` on the table  |
| IAM role        | `bug-report-tool-stack-gateway-role`      | Assumed by the AgentCore Gateway to invoke the Lambda on the model's behalf  |
| IAM role        | `bug-report-tool-stack-harness-role`      | Assumed by the managed harness to call Bedrock models and invoke the gateway |

Running `setup_gateway.py` registered the Lambda behind an AgentCore Gateway target named `bugreports`, so the model sees the tool as `bugreports___create_bug_report`.

#### Test the Lambda Function

Before wiring the tool into your prompt, verify it works in isolation. In the Lambda console, open the `bug-report-tool-stack-create-bug-report` function and go to the **Test** tab.

The AgentCore Gateway sends the tool arguments **directly as the Lambda event** — a plain JSON object with no wrapper envelope (Agents Classic used to wrap them in a `messageVersion`/`parameters` structure; the gateway does not). Create a new test event with the following JSON:

<pre class="css-0"><div data-defines-codeblock="true" tabindex="0" class="css-ift61f"><div class="css-1wj0762"></div><div><code class="language-json"><span class="token">{</span><span>
</span><span></span><span class="token">"description"</span><span class="token">:</span><span></span><span class="token">"The checkout page crashes when I click the Pay button"</span><span class="token">,</span><span>
</span><span></span><span class="token">"stepsToReproduce"</span><span class="token">:</span><span></span><span class="token">"1. Add an item to the cart. 2. Go to checkout. 3. Click Pay."</span><span class="token">,</span><span>
</span><span></span><span class="token">"environment"</span><span class="token">:</span><span></span><span class="token">"Chrome 120 on macOS Sonoma"</span><span>
</span><span></span><span class="token">}</span></code></div></div></pre>

![Screenshot of the Lambda console Test tab showing how to create a test event with the sample JSON payload](https://video.udacity-data.com/topher/2026/April/69cfe964_creating-test-event/creating-test-event.jpeg)

Creating a test event in the Lambda console

Click  **Test** . You should see a successful response containing a `ticketId` and `"status": "OPEN"`.

![Screenshot of the Lambda console showing a successful test execution with a response containing ticketId and status OPEN](https://video.udacity-data.com/topher/2026/April/69cfe994_successful-lambda-test/successful-lambda-test.jpeg)

Successful Lambda test result

To confirm the record was written, go to the **DynamoDB** console, open the `bug-report-tool-stack-bug-reports` table, and click  **Explore table items** . You should see one item with the ticket ID from the response.

![Screenshot of the DynamoDB console showing a bug report item in the bug-report-tool-stack-bug-reports table with ticketId, description, stepsToReproduce, environment, and status fields](https://video.udacity-data.com/topher/2026/April/69cfe9ac_dynamodb-item/dynamodb-item.jpeg)

Bug report record created in DynamoDB

> **Troubleshooting:** If the test fails with `AccessDeniedException`, check that the IAM policy is attached to the correct execution role. If it fails with `ResourceNotFoundException`, verify that the Lambda's `TABLE_NAME` environment variable matches the table name (`bug-report-tool-stack-bug-reports`). The Lambda also prints every event it receives to CloudWatch Logs (`/aws/lambda/bug-report-tool-stack-create-bug-report`) — the ground truth for what actually reached it.

### Step 2: Build the Harness — Design the System Prompt

Now the fun part. Open `system_prompt.txt` and write the system prompt for your chatbot. Your application needs to handle three types of requests, and **the routing between them is done entirely by your prompt** — there are no condition nodes or classifiers, just instructions:

* **Bug reports** — collect additional information and create a ticket using the `create_bug_report` tool from Step 1
* **Platform questions** — answer common questions about orders, shipping, returns, and payments using the FAQ
* **Other requests** — politely redirect the customer to a human support phone line

#### Bug Report Tool Parameters

The `create_bug_report` tool accepts three parameters — all of them required:

| Parameter            | Required | Description                                     |
| -------------------- | -------- | ----------------------------------------------- |
| `description`      | Yes      | Description of the bug, in the customer's words |
| `stepsToReproduce` | Yes      | Steps to follow to reproduce the issue          |
| `environment`      | Yes      | Customer's environment (browser, OS, device)    |

Customers rarely provide all three up front. Because the harness keeps **session state** across turns, your chatbot can simply *ask* for what's missing — make sure your prompt tells it to collect all three fields (and how to behave while collecting them, e.g. one question at a time) before calling the tool, and to give the customer their ticket ID afterwards.

#### Embed the FAQ

Platform questions (orders, shipping, returns, payments) need to be answered from the shop's FAQ. Here we use the simplest approach and embed the document directly in the prompt — the model sees it at inference time and answers from it. Keep the `{{FAQ}}` placeholder in `system_prompt.txt`; `create_harness.py` replaces it with the contents of `online_shop_faq.md` automatically.

> **Note:** Embedding documents in the prompt works well for short, stable content like a FAQ. For large documents, embedding the full text in every prompt becomes expensive and hits context limits. The standard solution is  **Retrieval-Augmented Generation (RAG)** , which retrieves only the relevant passages at query time using a vector index. RAG with Amazon Bedrock Knowledge Bases is outside the scope of this course.

#### Create the Harness and Iterate

When your prompt is ready, create the harness and chat with it:

<pre class="css-0"><div data-defines-codeblock="true" tabindex="0" class="css-ift61f"><div class="css-1wj0762"></div><div><code class="language-bash"><span>python create_harness.py     </span><span class="token"># first run takes ~2-3 minutes</span><span>
</span><span>python chat.py               </span><span class="token"># each run = one fresh conversation</span></code></div></div></pre>

Iterating is fast: edit `system_prompt.txt`, re-run `create_harness.py` (it updates the existing harness), and start a new `chat.py` session. There is no "prepare" step and nothing to redeploy — the harness picks up prompt changes as soon as `create_harness.py` finishes.

#### Tips

* Treat routing as a classification problem inside your prompt: describe the three categories crisply and tell the model to pick exactly one before doing anything else. Vague category definitions produce vague routing.
* Be explicit about the bug-report checklist (description, steps to reproduce, environment) and tell the model **not** to call the tool until every item is collected. Asking one question at a time works noticeably better than asking for everything at once.
* Tell the model to answer platform questions *only* from the FAQ, and what to do when the FAQ doesn't cover the question (that's the hand-off case).
* When the tool succeeds it returns a `ticketId` — instruct the model to relay it to the customer, so you can find the ticket in DynamoDB later.
* Verify tickets really land in the database: `aws dynamodb scan --table-name bug-report-tool-stack-bug-reports --region us-east-1`.
* The tool call appears in `chat.py` as a `[tool call] bugreports___create_bug_report` line — if you never see it, your prompt probably isn't telling the model clearly when to use the tool. (The `bugreports` prefix is the gateway target name; target names may only contain letters, digits, and underscores — no dashes.)
* Implement and test your solution step by step.
* Use the `us-east-1` region, as some smaller regions might not have all Bedrock AgentCore features.

### Step 3: Testing

Once your chatbot works, you can keep testing it manually with `chat.py`. However, manual testing is tedious and not scalable. For automated testing:

1. Create a set of test prompts and define expected results — copy `harness-tests-template.json` (e.g. to `harness-tests.json`) and fill in your test cases, covering all three routes
2. Run your application programmatically on the test set with `generate-eval-dataset.py`
3. Use Bedrock Evaluations to evaluate your application's outputs

Follow the detailed steps in the **Testing Framework** page to run automated tests and evaluate your chatbot.

### Submission Checklist

Your project will be evaluated on these criteria:

1. **Routing** — The system prompt routes every customer message to exactly one of three behaviors: bug-report collection, FAQ answering, or a polite human hand-off. The category definitions are crisp enough that routing is predictable across the test suite.
2. **Bug Report Handling** — The chatbot collects the bug description, the steps to reproduce, and the customer's environment across the conversation before calling the `create_bug_report` tool, then relays the ticket ID to the customer. A record is created in the `bug-report-tool-stack-bug-reports` DynamoDB table.
3. **FAQ and Hand-Off Handling** — Platform questions are answered *only* from the FAQ embedded via the `{{FAQ}}` placeholder (with a hand-off when the FAQ doesn't cover the question). Out-of-scope requests get a polite redirect to the human support phone line.
4. **Testing and Evaluation** — `harness-tests.json` covers all three routes. The `generate-eval-dataset.py` script produces a JSONL output. A Bedrock Evaluations job is created, and you provide written observations on the results.

### Stand-Out Suggestions

* Add edge-case test prompts: ambiguous messages, very short messages, prompt injection attempts
* Harden your system prompt against prompt injection (e.g. instructions that survive "ignore your previous instructions")
* Add multi-turn bug-report tests: script a full collection conversation in `chat.py` and verify the ticket fields in DynamoDB match what the customer said
* Extend the FAQ with your own entries and verify the chatbot picks up the new answers without any redeploy beyond re-running `create_harness.py`
