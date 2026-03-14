# MCP SDK - Prompts

> Official Documentation: https://modelcontextprotocol.io/docs/concepts/prompts

## Overview

Prompts are reusable message templates that servers expose to help structure LLM interactions. Prompts are **user-controlled**: they are selected explicitly by users (e.g., as slash commands in a chat UI). They can be parameterized with arguments to produce dynamic message sequences.

---

## Table of Contents

1. [Prompt Definition and Structure](#prompt-definition-and-structure)
2. [Capabilities](#capabilities)
3. [Registering Prompts with McpServer](#registering-prompts-with-mcpserver)
4. [Prompt Arguments (argsSchema)](#prompt-arguments)
5. [Dynamic Prompts](#dynamic-prompts)
6. [Multi-message Prompts](#multi-message-prompts)
7. [Prompts with Embedded Resources](#prompts-with-embedded-resources)
8. [Argument Completions](#argument-completions)
9. [Protocol Messages](#protocol-messages)
10. [Use Cases](#use-cases)
11. [Best Practices](#best-practices)

---

## Prompt Definition and Structure

A prompt definition includes:
- `name`: Unique identifier within the server
- `title`: Optional human-readable display name
- `description`: Optional human-readable description
- `arguments`: Optional list of arguments for customization

Each `PromptMessage` contains:
- `role`: `"user"` or `"assistant"`
- `content`: text, image, audio, or embedded resource

---

## Capabilities

Servers that support prompts must declare the `prompts` capability:

```typescript
const server = new McpServer(
    { name: 'my-server', version: '1.0.0' },
    {
        capabilities: {
            prompts: {
                listChanged: true  // server will notify when the prompt list changes
            }
        }
    }
);
```

---

## Registering Prompts with McpServer

Use `server.registerPrompt()` to register prompts on an `McpServer` instance.

### Minimal prompt (no arguments)
```typescript
import { McpServer } from '@modelcontextprotocol/server';
import { StdioServerTransport } from '@modelcontextprotocol/server';
import { z } from 'zod';

const server = new McpServer({ name: 'my-server', version: '1.0.0' });

server.registerPrompt(
    'summarize-conversation',
    {
        title: 'Summarize Conversation',
        description: 'Generate a concise summary of the current conversation'
    },
    () => ({
        messages: [
            {
                role: 'user' as const,
                content: {
                    type: 'text' as const,
                    text: 'Please summarize the key points from our conversation so far, organized by topic.'
                }
            }
        ]
    })
);

const transport = new StdioServerTransport();
await server.connect(transport);
```

### Prompt with arguments
```typescript
server.registerPrompt(
    'review-code',
    {
        title: 'Code Review',
        description: 'Review code for best practices and potential issues',
        argsSchema: z.object({
            code: z.string().describe('The code to review'),
            language: z.string().optional().describe('Programming language (e.g., TypeScript, Python)')
        })
    },
    ({ code, language }) => ({
        messages: [
            {
                role: 'user' as const,
                content: {
                    type: 'text' as const,
                    text: `Please review this ${language ?? 'code'} for best practices, potential bugs, and improvements:\n\n\`\`\`${language ?? ''}\n${code}\n\`\`\``
                }
            }
        ]
    })
);
```

---

## Prompt Arguments

The `argsSchema` uses Zod to define prompt arguments. The SDK converts this to the MCP argument list format.

### Arguments with validation
```typescript
server.registerPrompt(
    'explain-concept',
    {
        title: 'Explain Concept',
        description: 'Explain a technical concept at a specified level',
        argsSchema: z.object({
            concept: z.string().describe('The concept to explain'),
            level: z.enum(['beginner', 'intermediate', 'expert']).describe('Expertise level of the explanation'),
            includeExamples: z.boolean().optional().default(true).describe('Whether to include code examples')
        })
    },
    ({ concept, level, includeExamples }) => {
        const examplesInstruction = includeExamples
            ? 'Include practical code examples to illustrate key points.'
            : 'Focus on the conceptual explanation without code examples.';

        return {
            messages: [
                {
                    role: 'user' as const,
                    content: {
                        type: 'text' as const,
                        text: `Explain "${concept}" at a ${level} level. ${examplesInstruction}`
                    }
                }
            ]
        };
    }
);
```

---

## Dynamic Prompts

Prompts can fetch data dynamically to include in their messages.

```typescript
server.registerPrompt(
    'analyze-recent-errors',
    {
        title: 'Analyze Recent Errors',
        description: 'Analyze error logs from the last N hours',
        argsSchema: z.object({
            hours: z.number().min(1).max(72).default(24).describe('Number of hours to look back'),
            service: z.string().describe('Service name to analyze')
        })
    },
    async ({ hours, service }) => {
        // Fetch live data to include in the prompt
        const errors = await fetchRecentErrors(service, hours);
        const errorSummary = errors
            .map(e => `[${e.timestamp}] ${e.level}: ${e.message}`)
            .join('\n');

        return {
            messages: [
                {
                    role: 'user' as const,
                    content: {
                        type: 'text' as const,
                        text: `Analyze the following error logs from ${service} in the last ${hours} hours and identify patterns, root causes, and recommended fixes:\n\n${errorSummary}`
                    }
                }
            ]
        };
    }
);
```

---

## Multi-message Prompts

Prompts can include multiple messages with alternating roles to provide few-shot examples or structured conversations.

```typescript
server.registerPrompt(
    'sql-assistant',
    {
        title: 'SQL Assistant',
        description: 'Help write and optimize SQL queries with examples',
        argsSchema: z.object({
            task: z.string().describe('Description of what you want to query'),
            schema: z.string().optional().describe('Database schema information')
        })
    },
    ({ task, schema }) => {
        const messages = [];

        // System context via first user message
        messages.push({
            role: 'user' as const,
            content: {
                type: 'text' as const,
                text: 'You are an expert SQL developer. Help me write efficient, correct SQL queries.'
            }
        });
        messages.push({
            role: 'assistant' as const,
            content: {
                type: 'text' as const,
                text: "I'll help you write SQL queries. Please describe what you need."
            }
        });

        // Few-shot example
        messages.push({
            role: 'user' as const,
            content: {
                type: 'text' as const,
                text: 'Get the top 5 customers by total order value'
            }
        });
        messages.push({
            role: 'assistant' as const,
            content: {
                type: 'text' as const,
                text: '```sql\nSELECT c.name, SUM(o.amount) as total\nFROM customers c\nJOIN orders o ON c.id = o.customer_id\nGROUP BY c.id, c.name\nORDER BY total DESC\nLIMIT 5;\n```'
            }
        });

        // The actual request
        const contextText = schema ? `\nSchema:\n${schema}\n\n` : '';
        messages.push({
            role: 'user' as const,
            content: {
                type: 'text' as const,
                text: `${contextText}Task: ${task}`
            }
        });

        return { messages };
    }
);
```

---

## Prompts with Embedded Resources

Prompts can reference server-managed resources directly within their messages.

```typescript
server.registerPrompt(
    'review-file',
    {
        title: 'Review File',
        description: 'Review a specific file from the project',
        argsSchema: z.object({
            filePath: z.string().describe('Path to the file to review'),
            focus: z.enum(['security', 'performance', 'readability', 'all']).default('all')
        })
    },
    async ({ filePath, focus }) => {
        const fileContent = await readFile(filePath, 'utf-8');
        const focusText = focus === 'all'
            ? 'all aspects including security, performance, and readability'
            : focus;

        return {
            messages: [
                {
                    role: 'user' as const,
                    content: {
                        type: 'text' as const,
                        text: `Please review the following file for ${focusText}:`
                    }
                },
                {
                    role: 'user' as const,
                    content: {
                        // Embedded resource content
                        type: 'resource' as const,
                        resource: {
                            uri: `file://${filePath}`,
                            mimeType: detectMimeType(filePath),
                            text: fileContent
                        }
                    }
                }
            ]
        };
    }
);
```

### Including images in prompts
```typescript
server.registerPrompt(
    'describe-screenshot',
    {
        title: 'Describe Screenshot',
        description: 'Describe and analyze a screenshot',
        argsSchema: z.object({
            imagePath: z.string().describe('Path to the screenshot file')
        })
    },
    async ({ imagePath }) => {
        const imageData = await readFile(imagePath);
        return {
            messages: [
                {
                    role: 'user' as const,
                    content: {
                        type: 'text' as const,
                        text: 'Please describe what you see in this screenshot and identify any UI issues:'
                    }
                },
                {
                    role: 'user' as const,
                    content: {
                        type: 'image' as const,
                        data: imageData.toString('base64'),
                        mimeType: 'image/png'
                    }
                }
            ]
        };
    }
);
```

---

## Argument Completions

Use `completable()` to provide autocompletion suggestions for prompt arguments:

```typescript
import { completable } from '@modelcontextprotocol/server';

server.registerPrompt(
    'review-code',
    {
        title: 'Code Review',
        description: 'Review code for best practices',
        argsSchema: z.object({
            // Wrap with completable() to provide autocomplete
            language: completable(
                z.string().describe('Programming language'),
                (value) => ['typescript', 'javascript', 'python', 'rust', 'go', 'java']
                    .filter(lang => lang.startsWith(value.toLowerCase()))
            ),
            code: z.string().describe('The code to review')
        })
    },
    ({ language, code }) => ({
        messages: [
            {
                role: 'user' as const,
                content: {
                    type: 'text' as const,
                    text: `Review this ${language} code:\n\n\`\`\`${language}\n${code}\n\`\`\``
                }
            }
        ]
    })
);
```

---

## Protocol Messages

### Listing Prompts (prompts/list)
```json
// Request
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "prompts/list",
  "params": {
    "cursor": "optional-cursor-value"
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "prompts": [
      {
        "name": "review-code",
        "title": "Request Code Review",
        "description": "Asks the LLM to analyze code quality and suggest improvements",
        "arguments": [
          {
            "name": "code",
            "description": "The code to review",
            "required": true
          },
          {
            "name": "language",
            "description": "Programming language",
            "required": false
          }
        ]
      }
    ],
    "nextCursor": "next-page-cursor"
  }
}
```

### Getting a Prompt (prompts/get)
```json
// Request
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "prompts/get",
  "params": {
    "name": "review-code",
    "arguments": {
      "code": "def hello():\n    print('world')",
      "language": "python"
    }
  }
}

// Response
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "description": "Code review prompt",
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "Please review this python code:\n\n```python\ndef hello():\n    print('world')\n```"
        }
      }
    ]
  }
}
```

### List Changed Notification
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/prompts/list_changed"
}
```

### Error Responses
```json
// Invalid prompt name
{
  "jsonrpc": "2.0",
  "id": 3,
  "error": {
    "code": -32602,
    "message": "Invalid params: unknown prompt 'nonexistent-prompt'"
  }
}

// Missing required argument
{
  "jsonrpc": "2.0",
  "id": 4,
  "error": {
    "code": -32602,
    "message": "Invalid params: required argument 'code' is missing"
  }
}
```

---

## Use Cases

| Use Case | Description | Example |
|----------|-------------|---------|
| **Slash commands** | User-triggered commands in chat UIs | `/review`, `/summarize`, `/explain` |
| **Code review templates** | Standardized code analysis prompts | Review for security, performance, style |
| **Documentation generation** | Generate docs from code | API docs, README, changelog |
| **Data analysis** | Structured data exploration prompts | Analyze logs, metrics, database records |
| **Few-shot examples** | Provide examples before the actual request | SQL generation, test writing |
| **System prompts** | Pre-built system context for specific tasks | Customer support, technical writing |
| **Multi-step workflows** | Chain of thought prompting | Debugging, planning, research |

---

## Best Practices

1. **Clear, actionable names**: use `verb-noun` pattern — `review-code`, `analyze-logs`, `generate-tests`

2. **Informative descriptions**: the description is shown to users to help them choose the right prompt

3. **Validate arguments**: always validate inputs with Zod — use `.optional()` with `.default()` for non-required arguments

4. **Fetch data asynchronously**: prompts can be async — fetch relevant context dynamically to make prompts more useful

5. **Use multi-message for few-shot**: include example user/assistant exchanges before the actual request to guide the LLM

6. **Include embedded resources**: embed relevant files or data directly in the prompt rather than asking users to copy-paste

7. **Provide completions**: use `completable()` for arguments with known valid values to improve the user experience

8. **Handle errors gracefully**: validate arguments before processing; throw with descriptive messages for invalid inputs

9. **Use `listChanged`**: notify clients when prompts are added, removed, or modified

10. **Security**: validate all prompt inputs/outputs to prevent injection attacks; never include sensitive information in prompt templates
