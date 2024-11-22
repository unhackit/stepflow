
# Stepflow

A flexible, type-safe workflow automation library for Node.js.

[![npm version](https://img.shields.io/npm/v/@unhackit/stepflow.svg)](https://www.npmjs.com/package/@unhackit/stepflow)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![TypeScript](https://img.shields.io/badge/TypeScript-4.9+-blue.svg)](https://www.typescriptlang.org/)

## Features

🚀 **Core Capabilities**
- Step-based workflow execution
- Conditional branching
- Parallel execution
- Type-safe context passing

⚡ **Advanced Features**
- Input validation (Zod integration)
- Retry mechanisms with backoff strategies
- Built-in metrics collection
- Lifecycle hooks
- Error handling and recovery

## Installation

```bash
npm install @unhackit/stepflow
```

## Quick Start

```typescript
import { Workflow, WorkflowContext } from  "stepflow";

// Define the context shape for your workflow
interface OnboardingContext extends WorkflowContext {
  userId: string;
  email: string;
  userProfile?: {
    name: string;
    preferences: {
      theme: "light" | "dark";
      newsletter: boolean;
    };
  };
  welcomeEmailSent?: boolean;
  analyticsTracked?: boolean;
  error?: string;
}

const onboardingWorkflow = new Workflow<OnboardingContext>({
  name: "user-onboarding",
  hooks: {
    onError: ({ error, stepName }) => {
      console.error(`Error during ${stepName}:`, error);
    },
  },
}).addStep({
    fn: async ({ context }) => {
      context.userProfile = {
        name: context.email.split('@')[0],
        preferences: {
          theme: "light",
          newsletter: true,
        },
      };
    },
    config: { name: "create-profile" },
  }).addCondition({
	  branches: [
      {
        name: "newsletter-signup",
        condition: ({ context }) => context.userProfile?.preferences.newsletter === true,
        workflow: new Workflow<OnboardingContext>()
          .addStep({
            fn: async ({ context }) => {
              await subscribeToNewsletter(context.email);
            },
            config: {
              name: "subscribe-newsletter",
              retries: {
                maxAttempts: 3,
                backoff: { type: "exponential", delay: 1000 },
              },
            },
          }),
      },
    ],
  })
  .addStep({
    fn: async ({ context }) => {
      await sendWelcomeEmail(context.email, context.userProfile);
      context.welcomeEmailSent = true;
    },
    config: {
      name: "send-welcome-email",
      retries: { maxAttempts: 2, backoff: { type: "fixed", delay: 2000 } },
    },
  })
  .addStep({
    fn: async ({ context }) => {
      await trackSignup(context.userId, context.userProfile);
      context.analyticsTracked = true;
    },
    config: { name: "track-analytics" },
  });

const result = await onboardingWorkflow.execute({
  context: {
    userId: "user_123",
    email: "jane@example.com",
  },
});
```

## Core Concepts

### Conditional Branching

Create dynamic workflows with condition-based execution paths:

```typescript
workflow.addCondition({
  branches: [
    {
      name: "premium-user",
      condition: ({ context }) => context.userData?.isPremium,
      workflow: premiumWorkflow,
    },
    {
      name: "regular-user",
      condition: ({ context }) => !context.userData?.isPremium,
      workflow: regularWorkflow,
    },
  ],
});
```

### Parallel Execution

Run multiple workflows concurrently for improved performance:

```typescript
const mainWorkflow = new Workflow().parallel([
  notificationWorkflow,
  dataProcessingWorkflow,
  analyticsWorkflow,
]);
```

### Input Validation

Ensure data integrity with Zod schema validation:

```typescript
import { z } from "zod";
import { ZodValidator } from "@unhackit/stepflow";

const userSchema = z.object({
  userId: z.number(),
  email: z.string().email(),
  age: z.number().min(18),
});

const workflow = new Workflow({
  validator: new ZodValidator(userSchema),
});
```

### Error Handling

Implement robust error handling with lifecycle hooks:

```typescript
const workflow = new Workflow({
  hooks: {
    onError: ({ error, stepName, context }) => {
      console.error(`Error in step ${stepName}:`, error);
    },
  },
});
```

## API Reference

### Workflow

#### Constructor Options
| Option | Type | Description |
|--------|------|-------------|
| `name` | `string` | Optional workflow identifier |
| `validator` | `Validator<T>` | Input validation handler |
| `metricsCollector` | `MetricsCollector` | Custom metrics collection |
| `hooks` | `WorkflowHooks<T>` | Lifecycle event handlers |

#### Methods
| Method | Description |
|--------|-------------|
| `addStep(params: StepConstructorParams<T>)` | Add a new step to the workflow |
| `addCondition(config: ConditionConfig<T>)` | Add conditional branching |
| `parallel(workflows: Workflow<T>[])` | Execute workflows in parallel |
| `execute(params: { context: T })` | Run the workflow |

## Contributing

We welcome contributions! Please see our [Contributing Guide](CONTRIBUTING.md) for details on how to:
- Submit issues
- Create pull requests
- Follow our coding standards
- Join our community

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
