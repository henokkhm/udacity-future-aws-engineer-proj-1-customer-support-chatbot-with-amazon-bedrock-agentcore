# Customer Support Chatbot with Amazon Bedrock AgentCore

An agentic customer support chatbot built with the **Amazon Bedrock AgentCore** managed harness for a fictional online store. The chatbot autonomously triages customer messages across three distinct behaviors using a single, engineered system prompt and native Model Context Protocol (MCP) tool integration.

---

## Table of Contents

- [Customer Support Chatbot with Amazon Bedrock AgentCore](#customer-support-chatbot-with-amazon-bedrock-agentcore)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [Chatbot Behavior \& Routing](#chatbot-behavior--routing)
  - [Technology Stack](#technology-stack)
  - [Deployment](#deployment)
  - [License and Copyright](#license-and-copyright)
    - [Usage Restrictions](#usage-restrictions)

---

## Overview

This project implements an intelligent customer support assistant capable of routing incoming customer inquiries into one of three core paths:

1. **Bug Reports:** Interactively collects bug details (`description`, `stepsToReproduce`, and `environment`) across multiple conversation turns, then files a ticket in DynamoDB using the custom `create_bug_report` tool.
2. **Platform Questions:** Answers customer questions regarding orders, shipping, returns, and payments grounded strictly on an embedded FAQ document.
3. **Other Requests:** Politely redirects the customer to a human support phone line when inquiries fall outside the FAQ or bug-reporting scope.

> [!NOTE]
> **Why AgentCore?** Amazon Bedrock Agents Classic closed to new customers on July 30, 2026. This project uses its successor, the **AgentCore managed harness**, which provides server-side session memory, model invocation, and native MCP tool routing. Amazon Bedrock Evaluations is used for automated testing.

---

## Chatbot Behavior & Routing

The centerpiece of this project is prompt engineering. All message classification, multi-turn elicitation, and grounding logic reside in a single system prompt without separate classifier models or condition nodes:

- **Stateful Multi-Turn Elicitation:** Harness sessions retain conversational history across turns using `runtimeSessionId`. If a customer submits an incomplete bug report, the assistant prompts specifically for missing parameters before invoking tools.
- **Model Context Protocol (MCP) Tool Calling:** Once all parameters are collected, the model invokes `create_bug_report` through the AgentCore Gateway, which dispatches directly to AWS Lambda and records the ticket in Amazon DynamoDB.
- **In-Context Grounding:** Platform questions are answered exclusively from the embedded FAQ document, preventing hallucinations.
- **Fallback Escalation:** Out-of-scope or sensitive requests are politely escalated to the human customer support telephone line.

For complete deep-dive architectural diagrams and message flow sequences, refer to [**`Architecture.md`**](file:///home/henokkh/Desktop/Udacity%20Future%20AWS%20Agent%20Engineer/project_1/Architecture.md).

---

## Technology Stack

- **[Amazon Bedrock AgentCore Managed Harness](https://aws.amazon.com/bedrock/):** Runs the agent loop, stateful session memory, and tool execution.
- **Amazon Bedrock AgentCore Gateway:** Exposes the bug report Lambda as an MCP tool with `AWS_IAM` authentication.
- **[Amazon Bedrock Evaluations](https://aws.amazon.com/bedrock/):** Automated LLM-as-a-judge evaluation suite (Bring-Your-Own-Inference).
- **[AWS Lambda](https://aws.amazon.com/lambda/):** Serverless compute runtime executing the `create_bug_report` tool.
- **[Amazon DynamoDB](https://aws.amazon.com/dynamodb/):** NoSQL document database storing ticket records.
- **Amazon Nova Pro (`us.amazon.nova-pro-v1:0`):** Pinned foundation model using greedy decoding (`temperature=0.0`, `topK=1`).

---

## Deployment

Follow these steps to deploy the application and its underlying infrastructure to AWS:

**1: Deploy the tool stack** 

Deploy the CloudFormation template ([`cloudformation-tool.yaml`](./cloudformation-tool.yaml)) containing the DynamoDB bug reports table, the `create_bug_report` Lambda function, and the required IAM execution and gateway roles:

```bash
aws cloudformation deploy \
  --template-file cloudformation-tool.yaml \
  --stack-name bug-report-tool-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

**2: Create the gateway**

Run this once immediately after deploying the tool stack. [`setup_gateway.py`](./setup_gateway.py) reads the CloudFormation stack outputs, provisions an AgentCore Gateway (MCP protocol, `AWS_IAM` authentication), registers the Lambda function as the `create_bug_report` tool target, and saves all configuration ARNs to `agentcore_config.json`:

```bash
python setup_gateway.py
```

**3: Edit the system prompt**

Edit the  [`system_prompt.txt`](./system_prompt.txt) file to update the system prompt. **Note:** After every edit, you have to re-run  the following command to udate the deployed system prompt.

```bash
python create_harness.py
```

**4. Test the chat**

To chat with the agent in the CLI, run: 

```bash
python chat.py
```

**5. Automated Testing**

First add test cases to [`harness-test.json`](./harness-test.json)


Then generate the dataset: 

```bash
python generate-eval-dataset.py --tests-json harness-tests.json
head -1 output_eval_dataset.jsonl
```

Deploy the testing stack:

```bash
aws cloudformation deploy \
--template-file cloudformation-testing.yaml \
--stack-name bug-report-testing-stack \
--capabilities CAPABILITY_NAMED_IAM \
--region us-east-1
```

To see the stack outputs:

```bash
aws cloudformation describe-stacks --stack-name bug-report-testing-stack \
--query 'Stacks[0].Outputs' --output table --region us-east-1
```

**Note:** The above stack outputs are requred for running the next commands.

To run the evaluation:

```bash
aws s3 cp output_eval_dataset.jsonl s3://<BUCKET>/output_eval_dataset.jsonl --region us-east-1
```

Write the configs to files — replace <BUCKET> with the output of above stack:
```bash
python -c "
import json
json.dump({'automated':{'datasetMetricConfigs':[{'taskType':'General','dataset':
  {'name':'support-chatbot-eval-dataset','datasetLocation':
  {'s3Uri':'s3://<BUCKET>/output_eval_dataset.jsonl'}},
  'metricNames':['Builtin.Correctness']}],
  'evaluatorModelConfig':{'bedrockEvaluatorModels':
  [{'modelIdentifier':'amazon.nova-pro-v1:0'}]}}},
  open('eval-config.json','w'))
json.dump({'models':[{'precomputedInferenceSource':
  {'inferenceSourceIdentifier':'my-support-chatbot'}}]},
  open('inference-config.json','w'))
json.dump({'s3Uri':'s3://<BUCKET>/results/'}, open('output-config.json','w'))
"
```

Run the evaluation job: 

```bash
aws bedrock create-evaluation-job \
--job-name support-chatbot-eval-run-1 \
--role-arn <ROLE_ARN> \
--evaluation-config file://eval-config.json \
--inference-config file://inference-config.json \
--output-data-config file://output-config.json \
--region us-east-1
```

Note: For subsequent evaluation jobs, replace run-1 with the next number like run-2, run-3, etc

To check the status of all evaluations: 

```bash
aws bedrock list-evaluation-jobs --region us-east-1 \
--query 'jobSummaries[].[jobName,status]' --output table
```

Once the status of your latest evaluation is "Completed", you can see the results in the AWS Console -> Bedrock -> Evaluations -> <Your Evaluation>

**6. Clean up**

After completing the project, delete all resources not to incur additional costs. Run each command one by one:

```bash
python cleanup_agentcore.py
```

```bash
aws s3 rm s3://<BUCKET> --recursive --region us-east-1
```

```bash
aws cloudformation delete-stack --stack-name bug-report-testing-stack --region us-east-1
```

```bash
aws cloudformation delete-stack --stack-name bug-report-tool-stack --region us-east-1
```

```bash
rm -rf venv
```


---

## License and Copyright

- **Starter Code & Educational Material:** Copyright © 2012–2024 Udacity, Inc. Licensed under [CC BY-NC-ND 4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/).
- **Homework Solution / Student Modifications:** 👤 **Henok Kirubel Hailemariam**
  - GitHub: [@henokkhm](https://github.com/henokkhm)
  - LinkedIn: [LinkedIn](https://www.linkedin.com/in/henokkhm/)

### Usage Restrictions
This repository contains completed educational coursework based on Udacity starter materials. 
Because the starter content is subject to the CC BY-NC-ND 4.0 license, this repository is made 
public strictly for personal portfolio, educational demonstration, and peer-review purposes. 
No commercial use, redistribution, or further derivative works are permitted.