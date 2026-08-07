# MCP Server Guide: GitHub Issues Integration for Cline

## Overview

This document provides a detailed, step-by-step guide for building a custom MCP (Model Context Protocol) server that integrates GitHub Issues with Cline, the AI coding assistant. This MCP server enables Cline to automatically create, read, update, and delete GitHub Issues (tasks, stories, sub-tasks) as part of the development workflow.

---

## Table of Contents

1. [What is an MCP Server?](#1-what-is-an-mcp-server)
2. [Prerequisites](#2-prerequisites)
3. [Step 1: Set Up the Project Structure](#3-step-1-set-up-the-project-structure)
4. [Step 2: Initialize package.json](#4-step-2-initialize-packagejson)
5. [Step 3: Install Dependencies](#5-step-3-install-dependencies)
6. [Step 4: Create the MCP Server Entry Point](#6-step-4-create-the-mcp-server-entry-point)
7. [Step 5: Implement GitHub API Client](#7-step-5-implement-github-api-client)
8. [Step 6: Implement MCP Tools](#8-step-6-implement-mcp-tools)
9. [Step 7: Implement MCP Resources](#9-step-7-implement-mcp-resources)
10. [Step 8: Build and Test the Server](#10-step-8-build-and-test-the-server)
11. [Step 9: Register the MCP Server in Cline](#11-step-9-register-the-mcp-server-in-cline)
12. [Step 10: Create Initial GitHub Issues from the Plan](#12-step-10-create-initial-github-issues-from-the-plan)
13. [What to Expect at the End](#13-what-to-expect-at-the-end)
14. [Troubleshooting](#14-troubleshooting)
15. [Next Steps](#15-next-steps)

---

## 1. What is an MCP Server?

The **Model Context Protocol (MCP)** is an open protocol developed by Anthropic that allows AI assistants (like Cline) to securely connect to external data sources and tools. An MCP server is a process that:

- **Exposes Tools**: Functions that Cline can call (e.g., `create_issue`, `list_issues`)
- **Exposes Resources**: Data sources that Cline can read (e.g., `github://issues/123`)
- **Communicates via stdio**: MCP servers communicate with Cline over standard input/output using JSON-RPC

When you ask Cline to "create a GitHub issue for the member CRUD feature," Cline will call the `create_issue` tool exposed by this MCP server, which in turn calls the GitHub REST API.

---

## 2. Prerequisites

Before starting, ensure you have:

1. **Node.js** (v18 or later) — Already installed (used for the Angular project)
2. **npm** (v11.16.0) — Already installed
3. **GitHub Personal Access Token (PAT)** with the following scopes:
   - `repo` (full control of private repositories)
   - `read:org` (read organization data, if applicable)
4. **GitHub CLI** (optional, for testing) — Check with `gh --version`
5. **Cline** with MCP support — Already in use

### Creating a GitHub Personal Access Token

1. Go to https://github.com/settings/tokens
2. Click "Generate new token" → "Tokens (classic)"
3. Select scopes:
   - `repo` — Full control of private repositories
   - `read:org` — Read organization data (if your repo is under an org)
4. Click "Generate token"
5. Save the token securely — you'll need it as an environment variable

---

## 3. Step 1: Set Up the Project Structure

Create a new directory for the MCP server inside the project:

```bash
mkdir -p /Users/Farhad/sandbox/club/mcp-servers/github-issues
cd /Users/Farhad/sandbox/club/mcp-servers/github-issues
```

The directory structure will look like:

```
mcp-servers/github-issues/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts          # MCP server entry point
│   ├── github-client.ts  # GitHub API client wrapper
│   ├── tools.ts          # MCP tool definitions
│   └── resources.ts      # MCP resource definitions
└── .env                  # Environment variables (NOT committed to git)
```

---

## 4. Step 2: Initialize package.json

Run `npm init -y` to create a basic `package.json`, then update it:

```json
{
  "name": "mcp-server-github-issues",
  "version": "1.0.0",
  "description": "MCP server for GitHub Issues integration with Cline",
  "type": "module",
  "main": "dist/index.js",
  "bin": {
    "mcp-server-github-issues": "dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "ts-node src/index.ts"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^0.5.0",
    "axios": "^1.7.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "ts-node": "^10.9.0"
  }
}
```

---

## 5. Step 3: Install Dependencies

```bash
npm install
```

This installs:

- `@modelcontextprotocol/sdk` — The official MCP SDK for Node.js
- `axios` — HTTP client for making GitHub API requests
- `typescript` and `ts-node` — For TypeScript compilation and execution

---

## 6. Step 4: Create the MCP Server Entry Point

Create `src/index.ts` — this is the main entry point that sets up the MCP server:

```typescript
#!/usr/bin/env node
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListResourcesRequestSchema,
  ListToolsRequestSchema,
  ReadResourceRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';
import { GitHubClient } from './github-client.js';
import { tools } from './tools.js';
import { resources } from './resources.js';

// Read configuration from environment variables
const GITHUB_TOKEN = process.env.GITHUB_TOKEN;
const GITHUB_OWNER = process.env.GITHUB_OWNER || 'fbayanati';
const GITHUB_REPO = process.env.GITHUB_REPO || 'club-management-system';

if (!GITHUB_TOKEN) {
  console.error('Error: GITHUB_TOKEN environment variable is required');
  process.exit(1);
}

const client = new GitHubClient(GITHUB_TOKEN, GITHUB_OWNER, GITHUB_REPO);

// Create the MCP server
const server = new Server(
  {
    name: 'mcp-server-github-issues',
    version: '1.0.0',
  },
  {
    capabilities: {
      tools: {},
      resources: {},
    },
  },
);

// Register tool handler
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return { tools };
});

// Register resource handler
server.setRequestHandler(ListResourcesRequestSchema, async () => {
  return { resources };
});

// Register tool call handler
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  try {
    const tool = tools.find((t) => t.name === name);
    if (!tool) {
      throw new Error(`Unknown tool: ${name}`);
    }

    const result = await tool.handler(client, args || {});
    return {
      content: [
        {
          type: 'text',
          text: JSON.stringify(result, null, 2),
        },
      ],
    };
  } catch (error) {
    return {
      content: [
        {
          type: 'text',
          text: `Error: ${error instanceof Error ? error.message : String(error)}`,
        },
      ],
      isError: true,
    };
  }
});

// Register resource read handler
server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const { uri } = request.params;

  try {
    const result = await resources.read(client, uri);
    return {
      contents: [
        {
          uri,
          text: JSON.stringify(result, null, 2),
        },
      ],
    };
  } catch (error) {
    throw new Error(
      `Error reading resource ${uri}: ${error instanceof Error ? error.message : String(error)}`,
    );
  }
});

// Start the server
const transport = new StdioServerTransport();
await server.connect(transport);
```

---

## 7. Step 5: Implement GitHub API Client

Create `src/github-client.ts` — a wrapper around the GitHub REST API:

```typescript
import axios, { AxiosInstance } from 'axios';

export interface GitHubIssue {
  id: number;
  number: number;
  title: string;
  body: string | null;
  state: 'open' | 'closed';
  labels: Array<{ name: string; color: string }>;
  assignees: Array<{ login: string }>;
  milestone: { number: number; title: string } | null;
  created_at: string;
  updated_at: string;
  closed_at: string | null;
  html_url: string;
  user: { login: string };
}

export interface CreateIssueParams {
  title: string;
  body?: string;
  labels?: string[];
  assignees?: string[];
  milestone?: number;
}

export interface UpdateIssueParams {
  title?: string;
  body?: string;
  labels?: string[];
  assignees?: string[];
  milestone?: number | null;
  state?: 'open' | 'closed';
}

export class GitHubClient {
  private client: AxiosInstance;
  private owner: string;
  private repo: string;

  constructor(token: string, owner: string, repo: string) {
    this.owner = owner;
    this.repo = repo;
    this.client = axios.create({
      baseURL: 'https://api.github.com',
      headers: {
        Authorization: `token ${token}`,
        Accept: 'application/vnd.github+json',
        'X-GitHub-Api-Version': '2022-1128',
      },
    });
  }

  // List issues with optional filters
  async listIssues(
    options: {
      state?: 'open' | 'closed' | 'all';
      labels?: string;
      milestone?: string | number;
      assignee?: string;
      creator?: string;
      since?: string;
      per_page?: number;
      page?: number;
    } = {},
  ): Promise<GitHubIssue[]> {
    const params = new URLSearchParams();
    if (options.state) params.append('state', options.state);
    if (options.labels) params.append('labels', options.labels);
    if (options.milestone) params.append('milestone', String(options.milestone));
    if (options.assignee) params.append('assignee', options.assignee);
    if (options.creator) params.append('creator', options.creator);
    if (options.since) params.append('since', options.since);
    if (options.per_page) params.append('per_page', String(options.per_page));
    if (options.page) params.append('page', String(options.page));

    const response = await this.client.get<GitHubIssue[]>(
      `/repos/${this.owner}/${this.repo}/issues?${params.toString()}`,
    );
    return response.data;
  }

  // Get a single issue by number
  async getIssue(number: number): Promise<GitHubIssue> {
    const response = await this.client.get<GitHubIssue>(
      `/repos/${this.owner}/${this.repo}/issues/${number}`,
    );
    return response.data;
  }

  // Create a new issue
  async createIssue(params: CreateIssueParams): Promise<GitHubIssue> {
    const response = await this.client.post<GitHubIssue>(
      `/repos/${this.owner}/${this.repo}/issues`,
      params,
    );
    return response.data;
  }

  // Update an existing issue
  async updateIssue(number: number, params: UpdateIssueParams): Promise<GitHubIssue> {
    const response = await this.client.patch<GitHubIssue>(
      `/repos/${this.owner}/${this.repo}/issues/${number}`,
      params,
    );
    return response.data;
  }

  // Close an issue
  async closeIssue(number: number): Promise<GitHubIssue> {
    return this.updateIssue(number, { state: 'closed' });
  }

  // Reopen an issue
  async reopenIssue(number: number): Promise<GitHubIssue> {
    return this.updateIssue(number, { state: 'open' });
  }

  // Add a comment to an issue
  async addComment(number: number, body: string): Promise<any> {
    const response = await this.client.post(
      `/repos/${this.owner}/${this.repo}/issues/${number}/comments`,
      { body },
    );
    return response.data;
  }

  // List all labels
  async listLabels(): Promise<Array<{ name: string; color: string; description: string }>> {
    const response = await this.client.get(`/repos/${this.owner}/${this.repo}/labels`);
    return response.data;
  }

  // Create a label
  async createLabel(name: string, color: string, description?: string): Promise<any> {
    const response = await this.client.post(`/repos/${this.owner}/${this.repo}/labels`, {
      name,
      color,
      description,
    });
    return response.data;
  }

  // List all milestones
  async listMilestones(state: 'open' | 'closed' | 'all' = 'open'): Promise<any[]> {
    const response = await this.client.get(
      `/repos/${this.owner}/${this.repo}/milestones?state=${state}`,
    );
    return response.data;
  }

  // Create a milestone
  async createMilestone(title: string, description?: string, due_on?: string): Promise<any> {
    const response = await this.client.post(`/repos/${this.owner}/${this.repo}/milestones`, {
      title,
      description,
      due_on,
    });
    return response.data;
  }
}
```

---

## 8. Step 6: Implement MCP Tools

Create `src/tools.ts` — defines the tools that Cline can call:

```typescript
import { Tool } from '@modelcontextprotocol/sdk/types.js';
import { GitHubClient, CreateIssueParams, UpdateIssueParams } from './github-client.js';

export const tools: Tool[] = [
  {
    name: 'list_issues',
    description: 'List GitHub issues with optional filters (state, labels, milestone, assignee)',
    inputSchema: {
      type: 'object',
      properties: {
        state: { type: 'string', enum: ['open', 'closed', 'all'], description: 'Filter by state' },
        labels: { type: 'string', description: 'Comma-separated label names' },
        milestone: { type: 'string', description: 'Milestone number or "none"' },
        assignee: { type: 'string', description: 'GitHub username' },
        per_page: { type: 'number', description: 'Results per page (max 100)', default: 30 },
        page: { type: 'number', description: 'Page number', default: 1 },
      },
    },
    handler: async (client: GitHubClient, args: any) => {
      return await client.listIssues({
        state: args.state,
        labels: args.labels,
        milestone: args.milestone,
        assignee: args.assignee,
        per_page: args.per_page,
        page: args.page,
      });
    },
  },
  {
    name: 'get_issue',
    description: 'Get details of a specific GitHub issue by number',
    inputSchema: {
      type: 'object',
      properties: {
        number: { type: 'number', description: 'Issue number' },
      },
      required: ['number'],
    },
    handler: async (client: GitHubClient, args: any) => {
      return await client.getIssue(args.number);
    },
  },
  {
    name: 'create_issue',
    description: 'Create a new GitHub issue (task, story, bug, etc.)',
    inputSchema: {
      type: 'object',
      properties: {
        title: { type: 'string', description: 'Issue title' },
        body: { type: 'string', description: 'Issue body (markdown)' },
        labels: { type: 'array', items: { type: 'string' }, description: 'Labels to apply' },
        assignees: {
          type: 'array',
          items: { type: 'string' },
          description: 'GitHub usernames to assign',
        },
        milestone: { type: 'number', description: 'Milestone number' },
      },
      required: ['title'],
    },
    handler: async (client: GitHubClient, args: any) => {
      return await client.createIssue({
        title: args.title,
        body: args.body,
        labels: args.labels,
        assignees: args.assignees,
        milestone: args.milestone,
      });
    },
  },
  {
    name: 'update_issue',
    description: 'Update an existing GitHub issue (title, body, labels, state, etc.)',
    inputSchema: {
      type: 'object',
      properties: {
        number: { type: 'number', description: 'Issue number' },
        title: { type: 'string', description: 'New title' },
        body: { type: 'string', description: 'New body' },
        labels: { type: 'array', items: { type: 'string' }, description: 'Labels to set' },
        assignees: { type: 'array', items: { type: 'string' }, description: 'Assignees to set' },
        milestone: {
          type: 'number',
          nullable: true,
          description: 'Milestone number or null to remove',
        },
        state: { type: 'string', enum: ['open', 'closed'], description: 'Issue state' },
      },
      required: ['number'],
    },
    handler: async (client: GitHubClient, args: any) => {
      return await client.updateIssue(args.number, {
        title: args.title,
        body: args.body,
        labels: args.labels,
        assignees: args.assignees,
        milestone: args.milestone,
        state: args.state,
      });
    },
  },
  {
    name: 'close_issue',
    description: 'Close a GitHub issue by number',
    inputSchema: {
      type: 'object',
      properties: {
        number: { type: 'number', description: 'Issue number' },
      },
      required: ['number'],
    },
    handler: async (client: GitHubClient, args: any) => {
      return await client.closeIssue(args.number);
    },
  },
  {
    name: 'add_comment',
    description: 'Add a comment to a GitHub issue',
    inputSchema: {
      type: 'object',
      properties: {
        number: { type: 'number', description: 'Issue number' },
        body: { type: 'string', description: 'Comment body (markdown)' },
      },
      required: ['number', 'body'],
    },
    handler: async (client: GitHubClient, args: any) => {
      return await client.addComment(args.number, args.body);
    },
  },
  {
    name: 'list_labels',
    description: 'List all labels in the repository',
    inputSchema: {
      type: 'object',
      properties: {},
    },
    handler: async (client: GitHubClient, args: any) => {
      return await client.listLabels();
    },
  },
  {
    name: 'create_label',
    description: 'Create a new label in the repository',
    inputSchema: {
      type: 'object',
      properties: {
        name: { type: 'string', description: 'Label name' },
        color: { type: 'string', description: 'Hex color code (without #)' },
        description: { type: 'string', description: 'Label description' },
      },
      required: ['name', 'color'],
    },
    handler: async (client: GitHubClient, args: any) => {
      return await client.createLabel(args.name, args.color, args.description);
    },
  },
  {
    name: 'list_milestones',
    description: 'List all milestones in the repository',
    inputSchema: {
      type: 'object',
      properties: {
        state: {
          type: 'string',
          enum: ['open', 'closed', 'all'],
          description: 'Filter by state',
          default: 'open',
        },
      },
    },
    handler: async (client: GitHubClient, args: any) => {
      return await client.listMilestones(args.state || 'open');
    },
  },
  {
    name: 'create_milestone',
    description: 'Create a new milestone in the repository',
    inputSchema: {
      type: 'object',
      properties: {
        title: { type: 'string', description: 'Milestone title' },
        description: { type: 'string', description: 'Milestone description' },
        due_on: { type: 'string', description: 'Due date (YYYY-MM-DD)' },
      },
      required: ['title'],
    },
    handler: async (client: GitHubClient, args: any) => {
      return await client.createMilestone(args.title, args.description, args.due_on);
    },
  },
];
```

---

## 9. Step 7: Implement MCP Resources

Create `src/resources.ts` — defines resources that Cline can read:

```typescript
import { ListResourcesResult, ReadResourceResult } from '@modelcontextprotocol/sdk/types.js';
import { GitHubClient } from './github-client.js';

export const resources = {
  list: async (): Promise<ListResourcesResult> => {
    return {
      resources: [
        {
          uri: 'github://repo',
          name: 'Repository Info',
          description: 'Basic information about the GitHub repository',
          mimeType: 'application/json',
        },
        {
          uri: 'github://issues',
          name: 'All Open Issues',
          description: 'List of all open issues in the repository',
          mimeType: 'application/json',
        },
        {
          uri: 'github://milestones',
          name: 'All Milestones',
          description: 'List of all milestones in the repository',
          mimeType: 'application/json',
        },
        {
          uri: 'github://labels',
          name: 'All Labels',
          description: 'List of all labels in the repository',
          mimeType: 'application/json',
        },
      ],
    };
  },

  read: async (client: GitHubClient, uri: string): Promise<any> => {
    if (uri === 'github://repo') {
      // Return repository info
      const response = await (client as any).client.get(
        `/repos/${(client as any).owner}/${(client as any).repo}`,
      );
      return response.data;
    }

    if (uri === 'github://issues') {
      return await client.listIssues({ state: 'open' });
    }

    if (uri === 'github://milestones') {
      return await client.listMilestones('all');
    }

    if (uri === 'github://labels') {
      return await client.listLabels();
    }

    // Handle dynamic issue URIs: github://issues/{number}
    const issueMatch = uri.match(/^github:\/\/issues\/(\d+)$/);
    if (issueMatch) {
      const number = parseInt(issueMatch[1], 10);
      return await client.getIssue(number);
    }

    throw new Error(`Unknown resource URI: ${uri}`);
  },
};
```

---

## 10. Step 8: Build and Test the Server

### Build the server:

```bash
npm run build
```

### Test the server manually:

```bash
# Set environment variables
export GITHUB_TOKEN="your_personal_access_token_here"
export GITHUB_OWNER="fbayanati"
export GITHUB_REPO="club-management-system"

# Run the server
node dist/index.js
```

You should see no output (the server runs silently, waiting for MCP requests via stdio).

### Test with a simple MCP client:

You can test the server by piping a JSON-RPC request to it:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}' | node dist/index.js
```

This should return a JSON response listing all available tools.

---

## 11. Step 9: Register the MCP Server in Cline

### Option A: Using Cline's config file

1. Open (or create) the Cline MCP configuration file:
   - **macOS**: `~/Library/Application Support/Code/User/globalStorage/saoudrizwan.claude-dev/settings/cline_mcp_settings.json`
   - **Windows**: `%APPDATA%\Code\User\globalStorage\saoudrizwan.claude-dev\settings\cline_mcp_settings.json`
   - **Linux**: `~/.config/Code/User/globalStorage/saoudrizwan.claude-dev/settings/cline_mcp_settings.json`

2. Add the MCP server configuration:

```json
{
  "mcpServers": {
    "github-issues": {
      "command": "node",
      "args": ["/Users/Farhad/sandbox/club/mcp-servers/github-issues/dist/index.js"],
      "env": {
        "GITHUB_TOKEN": "your_personal_access_token_here",
        "GITHUB_OWNER": "fbayanati",
        "GITHUB_REPO": "club-management-system"
      }
    }
  }
}
```

### Option B: Using Cline's UI

1. Open Chone in VS Code
2. Click the Cline icon in the Activity Bar
3. Click the gear/settings icon
4. Go to "MCP Servers" tab
5. Click "Add MCP Server"
6. Enter:
   - **Name**: `github-issues`
   - **Command**: `node /Users/Farhad/sandbox/club/mcp-servers/github-issues/dist/index.js`
   - **Environment Variables**: Add `GITHUB_TOKEN`, `GITHUB_OWNER`, `GITHUB_REPO`

### Verify the registration:

After adding the server, restart VS Code. When you open Cline, it should detect the MCP server and list the available tools. You can verify by asking Cline:

> "What tools are available from the github-issues MCP server?"

---

## 12. Step 10: Create Initial GitHub Issues from the Plan

Once the MCP server is registered, Cline can automatically create GitHub Issues from the CLUB_MANAGEMENT_PLAN.md. Here's the workflow:

### 1. Create Labels

First, create the necessary labels:

```bash
# Cline can do this by calling the create_label tool:
# create_label(name="phase-1", color="1f7663", description="Phase 1: Foundation")
# create_label(name="phase-2", color="0969da", description="Phase 2: Core Features")
# create_label(name="phase-3", color="8250df", description="Phase 3: High-Value Additions")
# create_label(name="phase-4", color="bf8700", description="Phase 4: Operational Excellence")
# create_label(name="ugs", color="a4151c", description="User Goal/Story")
# create_label(name="task", color="1a7f37", description="Task")
# create_label(name="subtask", color="5389ce", description="Sub-task")
# create_label(name="bug", color="cf222e", description="Bug")
# create_label(name="enhancement", color="6f42c1", description="Enhancement")
```

### 2. Create Milestones

Create milestones for each phase:

```bash
# create_milestone(title="Phase 1: Foundation", description="Week 1 - Project setup, auth, layout, data layer")
# create_milestone(title="Phase 2: Core Features", description="Weeks 2-3 - Members, events, venues, invoices")
# create_milestone(title="Phase 3: High-Value Additions", description="Weeks 4-5 - Self-service, renewals, waitlists, QR")
# create_milestone(title="Phase 4: Operational Excellence", description="Week 6 - Permissions, retention, testing, deploy")
```

### 3. Create UGS (User Goal/Story) Issues

Cline can read the CLUB_MANAGEMENT_PLAN.md and create UGS issues for each major feature area:

- UGS-001: Member Management
- UGS-002: Event Management
- UGS-003: Financial Tracking (Invoices & Payments)
- UGS-004: Venue Management
- UGS-005: Communication & Engagement
- UGS-006: Reporting & Analytics
- UGS-007: Authentication & Authorization
- UGS-008: Dashboard

### 4. Create Task Issues

For each UGS, Cline creates child task issues with detailed descriptions, steps, and acceptance criteria.

### 5. Create Sub-task Issues

For each task, Cline can create sub-tasks for granular implementation steps.

---

## 13. What to Expect at the End

After completing all steps, you will have:

### 1. A Working MCP Server

- Located at `/Users/Farhad/sandbox/club/mcp-servers/github-issues/`
- Compiled and ready to run
- Exposes 10 tools and 5 resources for GitHub Issues management

### 2. Cline Integration

- Cline can now interact with GitHub Issues using natural language
- Example commands that will work:
  - "Create a GitHub issue for implementing the member list view with AG Grid"
  - "List all open issues labeled 'task' in the Phase 2 milestone"
  - "Close issue #15 and add a comment about the implementation"
  - "What are the acceptance criteria for issue #8?"

### 3. Automated Project Tracking

- All features from CLUB_MANAGEMENT_PLAN.md are tracked as GitHub Issues
- Issues are organized by phase (milestones) and feature area (labels)
- Cline can update issue status as it completes work
- Commit messages can reference issue numbers for traceability

### 4. UGS (User Goals/Stories) System

- Each major feature is a UGS issue with detailed goal, acceptance criteria, and solution approach
- Tasks are linked to UGS issues
- Sub-tasks break down tasks into implementable steps
- Cline can read UGS details to understand context before implementing

### 5. Ongoing Workflow

- As Cline implements features, it can:
  - Read task issues to understand requirements
  - Create new issues for bugs or enhancements discovered during development
  - Update issue status (open/closed) as work progresses
  - Add comments with implementation notes
  - Reference issues in commit messages

---

## 14. Troubleshooting

### MCP Server Not Detected by Cline

1. Check that the `dist/index.js` file exists (run `npm run build`)
2. Verify the path in `cline_mcp_settings.json` is correct
3. Ensure `GITHUB_TOKEN` is set and valid
4. Restart VS Code after changing the config

### GitHub API Rate Limit Errors

- GitHub allows 5,000 requests/hour for authenticated requests
- If you hit the limit, wait up to 1 hour for reset
- Consider caching responses in the MCP server for frequently accessed data

### Authentication Errors

- Ensure the PAT has the `repo` scope
- Verify the token hasn't expired
- Check that the token has access to the repository

### TypeScript Compilation Errors

- Ensure all imports use `.js` extensions (ESM requirement)
- Check that `@modelcontextprotocol/sdk` is installed
- Run `npm run build` and fix any errors before testing

---

## 15. Next Steps

After the MCP server is set up and registered:

1. **Create initial issues** — Use Cline to create all Phase 1-4 issues from the plan
2. **Start implementing** — Toggle to Act Mode and begin Phase 1 implementation
3. **Update as you go** — Cline will update issue status as it completes tasks
4. **Expand the MCP server** — Add more tools as needed (e.g., `search_issues`, `create_pull_request`, `list_branches`)
5. **Add automation** — Set up GitHub Actions to auto-close issues when PRs are merged

---

_Document generated: August 6, 2026_
_Project: Club Management System_
_Purpose: MCP Server Guide for GitHub Issues Integration_
