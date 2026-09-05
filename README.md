# Customer Support Chatbot with Amazon Bedrock AgentCore

An agentic customer support chatbot built with the **Amazon Bedrock AgentCore** managed harness for a fictional online store. The chatbot autonomously triages customer messages across three distinct behaviors using a single, engineered system prompt and native Model Context Protocol (MCP) tool integration.

---

## Table of Contents

- [Overview](#overview)
- [Chatbot Behavior & Routing](#chatbot-behavior--routing)
- [Technology Stack](#technology-stack)
- [License and Copyright](#license-and-copyright)

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