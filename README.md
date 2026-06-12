# AI Code Orchestrator Agent - System Architecture

## 1. Project Overview

The goal of this project is to build a **human-in-the-loop AI coding orchestration system**. Instead of the developer manually communicating with multiple AI tools, the developer communicates with one main AI agent. This agent coordinates with large language models such as Claude and ChatGPT, asks Claude Code to inspect the codebase, generates an implementation plan, and waits for human approval before any code execution happens.

The system is designed to help developers turn high-level requests into codebase-aware implementation plans and safe execution prompts for Claude Code.

Example user request:

> "Add authentication to this repository. Inspect the codebase first, create a plan, and wait for my approval before making changes."

The orchestrator agent handles the back-and-forth communication between models, retrieves context from the codebase, produces a final plan, and only generates an execution prompt after the user approves the plan.

---

## 2. Main Idea

The system follows this principle:

> The user talks to one AI agent. The AI agent talks to Claude, ChatGPT, and Claude Code on the user's behalf.

The orchestrator agent acts as the central controller. It decides when to ask Claude or ChatGPT for reasoning, when to ask Claude Code for codebase context, and when to return a final plan to the user.

This creates a safer and more structured workflow than directly asking an AI coding tool to modify the repository.

---

## 3. High-Level Architecture

```text
User
  |
  v
Web App / Chat Interface
  |
  v
AI Orchestrator Agent
  |
  +--> Model Router
  |      |
  |      +--> Claude API
  |      +--> OpenAI / ChatGPT API
  |      +--> Optional Reviewer / Critic Model
  |
  +--> Claude Code Worker
  |      |
  |      +--> Repository Sandbox
  |      +--> Codebase Inspection
  |      +--> File and Dependency Analysis
  |
  +--> Planning Engine
  |
  +--> Human Approval Layer
  |
  v
Final Plan + Claude Code Execution Prompt
```

---

## 4. Core Components

### 4.1 User Interface

The user interface can be a web app or command-line interface. The user submits a high-level software engineering request and receives a structured plan.

The interface should support:

- Submitting a task or bug request
- Selecting a GitHub repository or local codebase
- Viewing codebase findings
- Reviewing the implementation plan
- Approving, rejecting, or requesting revisions
- Viewing the final Claude Code execution prompt

Example user inputs:

```text
Fix the file upload API.
Add tests for the authentication flow.
Refactor the dashboard page.
Find why the app fails during deployment.
```

---

### 4.2 AI Orchestrator Agent

The AI Orchestrator Agent is the brain of the system. The user does not directly manage Claude, ChatGPT, or Claude Code. The orchestrator handles that automatically.

Responsibilities:

- Understand the user's request
- Decide what information is needed from the codebase
- Ask Claude Code to inspect relevant files
- Ask Claude and/or ChatGPT to reason about possible solutions
- Compare and refine proposed plans
- Ask follow-up questions when needed
- Produce a final plan for human review
- Generate a Claude Code execution prompt only after approval

The orchestrator should not directly modify code in the early version of the project.

---

### 4.3 Model Router

The Model Router decides which model should handle each task.

Possible routing strategy:

| Task | Preferred Tool / Model |
|---|---|
| Understand user request | ChatGPT or Claude |
| Inspect repository | Claude Code |
| Analyze architecture | Claude or ChatGPT |
| Generate implementation options | Claude and ChatGPT |
| Critique the plan | Reviewer model |
| Generate final execution prompt | Orchestrator agent |

This allows the system to use each model for what it does best.

---

### 4.4 Claude Code Worker

Claude Code is used as the codebase-aware worker. Its job is to inspect the repository and answer questions about the actual code.

The orchestrator may ask Claude Code questions such as:

```text
What framework is this project using?
Where are the API routes located?
Which files are related to authentication?
What tests currently exist?
What files are likely to change for this request?
Are there any risky dependencies or patterns?
```

In the first version, Claude Code should run in read-only inspection mode as much as possible. It should not edit files until the user approves the plan.

---

### 4.5 Repository Sandbox

The repository sandbox is an isolated environment where the codebase can be cloned and inspected safely.

The sandbox should protect the user from accidental or unsafe actions.

Important rules:

- Do not run destructive commands
- Do not access secrets or `.env` values
- Do not push directly to the main branch
- Do not modify files before user approval
- Use a temporary cloned repository
- Log all actions taken by the AI system

Recommended early command allowlist:

```text
ls
find
tree
cat
sed
grep
rg
git status
git diff
git log
npm test
pytest
```

Commands to avoid or block:

```text
rm
curl
wget
ssh
scp
aws
gh auth
docker run --privileged
chmod 777
```

---

### 4.6 Planning Engine

The Planning Engine combines the user's request, codebase findings, and model reasoning into a structured implementation plan.

The final plan should include:

- Summary of the user's request
- Codebase findings
- Relevant files
- Current behavior
- Proposed solution
- Step-by-step implementation plan
- Files likely to change
- Test plan
- Risks and assumptions
- Open questions
- Claude Code execution prompt

The system should clearly separate the **plan** from the **execution prompt**.

---

### 4.7 Human Approval Layer

The human approval layer is one of the most important parts of the system.

The system should not execute code changes automatically. It should first show the plan to the user and wait for a decision.

Possible user actions:

| Action | Meaning |
|---|---|
| Approve | Generate or run the Claude Code execution prompt |
| Revise | Ask the orchestrator to improve the plan |
| Ask Question | Request clarification before approving |
| Reject | Stop the workflow |

This makes the project safer and more realistic for professional software development.

---

## 5. Workflow

### Step 1: User submits a request

The user provides a high-level request through the chat interface.

Example:

```text
Add user authentication to this repository. Inspect the codebase first and give me a plan.
```

### Step 2: Orchestrator analyzes the request

The orchestrator identifies what it needs to know before planning.

It may determine that it needs to inspect:

- Project framework
- Routing structure
- Database layer
- Existing authentication code
- Middleware
- Test setup

### Step 3: Claude Code inspects the codebase

The orchestrator asks Claude Code to inspect the repository and return structured findings.

Example output:

```text
Framework: Next.js
Database: Prisma
API routes: app/api
Authentication: No existing auth system found
Testing: Jest is configured but auth tests do not exist
Relevant files:
- prisma/schema.prisma
- app/api/*
- middleware.ts
- package.json
```

### Step 4: Models generate and critique a plan

The orchestrator sends the codebase findings to Claude and/or ChatGPT. One model creates an initial plan. Another model reviews the plan and identifies risks.

Example critique questions:

```text
Is this plan too broad?
Are there security risks?
Are database migrations required?
Are there simpler alternatives?
What tests should be added?
```

### Step 5: Orchestrator creates final plan

The orchestrator combines the discussion into a final plan for the user.

### Step 6: User reviews the plan

The user can approve, reject, or request changes.

### Step 7: Generate Claude Code execution prompt

After approval, the system creates a precise prompt for Claude Code.

Example prompt:

```text
You are Claude Code working inside this repository.

Goal:
Implement the approved authentication plan.

Rules:
- Do not push to main.
- Create a new branch.
- Do not modify unrelated files.
- Do not access .env values.
- Show a diff before finalizing.
- Run tests after changes.
- If unsure, stop and ask.

Approved plan:
...

Relevant files:
...

Expected output:
- Summary of changes
- Files changed
- Commands run
- Test results
- Remaining risks
```

---

## 6. AWS Deployment Architecture

The system can be deployed on AWS to demonstrate cloud and AI engineering skills.

```text
User Browser
  |
  v
Frontend Web App
  |
  v
Backend API
  |
  v
Orchestrator Service
  |
  +--> Model APIs
  |      +--> OpenAI API
  |      +--> Anthropic Claude API
  |      +--> Amazon Bedrock models, optional
  |
  +--> Job Queue
  |      +--> Codebase Worker
  |              |
  |              +--> Temporary Repo Sandbox
  |
  +--> Database
  |
  +--> Logs and Monitoring
```

Recommended AWS services:

| Component | AWS Service Option |
|---|---|
| Frontend | AWS Amplify, S3 + CloudFront |
| Backend API | ECS Fargate, App Runner, or Lambda |
| Codebase Worker | ECS Fargate |
| Queue | Amazon SQS |
| Database | DynamoDB or RDS |
| File Storage | Amazon S3 |
| Secrets | AWS Secrets Manager |
| Logs | Amazon CloudWatch |
| Authentication | Amazon Cognito |
| Model Access | Anthropic API, OpenAI API, or Amazon Bedrock |

For this project, ECS Fargate is a good option for the codebase worker because repository cloning and command execution are easier in a container than in a short-lived Lambda function.

---

## 7. Data Flow

```text
1. User submits request
2. Backend stores request and creates planning job
3. Orchestrator asks Claude Code Worker to inspect repo
4. Worker returns codebase findings
5. Orchestrator sends findings to reasoning models
6. Reasoning models propose and critique plans
7. Orchestrator generates final plan
8. User reviews plan
9. If approved, orchestrator generates Claude Code execution prompt
10. Optional execution worker applies changes in a branch or produces a patch
```

---

## 8. Safety and Control Requirements

This project should be designed as a safe, human-in-the-loop system.

Key safety requirements:

- Never execute code changes without user approval
- Never push directly to the main branch
- Never expose secrets to model prompts
- Run code inspection in a sandbox
- Keep logs of model decisions and tool calls
- Limit the number of model-to-model discussion rounds
- Require confirmation before running tests, installing packages, or creating a pull request
- Show the user a final plan before execution

---

## 9. MVP Scope

The minimum viable product should focus on planning, not automatic code execution.

### MVP Features

- User submits a request
- User provides a GitHub repo URL or local repo path
- System inspects the codebase
- System identifies relevant files
- System generates a structured implementation plan
- System generates a Claude Code execution prompt
- User approves or requests revisions

### Not in MVP

- Fully automatic code modification
- Automatic pull request creation
- Production-level permission management
- Multi-user team collaboration
- Full CI/CD integration

---

## 10. Future Improvements

Possible future features:

- GitHub pull request integration
- Automatic branch creation after approval
- Test execution and result summarization
- Pull request risk scoring
- Cost tracking per planning session
- Model comparison dashboard
- Prompt history and plan versioning
- Support for multiple repositories
- Integration with Jira, Linear, or GitHub Issues
- RAG memory for previous codebase decisions

---

## 11. Summary

This system allows the developer to communicate with a single AI orchestrator instead of manually managing multiple AI tools. The orchestrator gathers codebase context through Claude Code, uses Claude and ChatGPT for reasoning and critique, and produces a final implementation plan for human review.

The most important design principle is safety: the system should plan first, ask for approval, and only execute after the user confirms.

This makes the project practical, professional, and suitable as a portfolio project for AI engineering, cloud engineering, and software development roles.
