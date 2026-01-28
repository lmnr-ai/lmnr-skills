# TypeScript/JavaScript instrumentation

## Initialize once

Place `Laminar.initialize()` as early as possible (after other instrumentation libraries).
For most semi-automatic instrumentations to work, you must pass `instrumentModules`.

```ts
import { Laminar } from '@lmnr-ai/lmnr';
import OpenAI from 'openai';
import Anthropic from '@anthropic-ai/sdk';

Laminar.initialize({
  projectApiKey: process.env.LMNR_PROJECT_API_KEY,
  instrumentModules: {
    openAI: OpenAI,
    anthropic: Anthropic,
  },
  // For self-hosted Laminar:
  // baseUrl: 'http://localhost',
  // httpPort: 8000,
  // grpcPort: 8001,
});
```

If a module is imported before `Laminar.initialize()` runs (common in Next.js server components),
use `Laminar.patch({ ... })` in the module that constructs the client.

```ts
import { Laminar } from '@lmnr-ai/lmnr';
import OpenAI from 'openai';

Laminar.patch({ openAI: OpenAI });
```

AI SDK (Vercel) instrumentation is **manual**: pass the Laminar tracer to
`experimental_telemetry` on each call.

```ts
import { getTracer } from '@lmnr-ai/lmnr';
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';

await generateText({
  model: openai('gpt-4.1-nano'),
  prompt: 'What is Laminar?',
  experimental_telemetry: {
    isEnabled: true,
    tracer: getTracer(),
  },
});
```

## Using another OpenTelemetry SDK

If you already use another OpenTelemetry setup (for example, Next.js with
`@vercel/otel`), pick one of these patterns:

**A) Single pipeline (export spans to Laminar via span processor).**  
Use `LaminarSpanProcessor` and `initializeLaminarInstrumentations()` inside your
existing OpenTelemetry init. Do **not** call `Laminar.initialize()` to avoid
double instrumentation. Spans will go to whatever exporters your OTel setup uses;
including `LaminarSpanProcessor` sends a copy to Laminar.

```ts
import { registerOTel } from '@vercel/otel';

export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    const {
      LaminarSpanProcessor,
      initializeLaminarInstrumentations,
    } = await import('@lmnr-ai/lmnr');

    registerOTel({
      serviceName: 'my-service',
      spanProcessors: [new LaminarSpanProcessor()],
      instrumentations: initializeLaminarInstrumentations(),
    });
  }
}
```

**B) Dual pipeline (general observability elsewhere, LLM tracing in Laminar).**  
Initialize your existing OTel SDK first, then call `Laminar.initialize()` to
instrument only the LLM SDKs you want in Laminar. Make sure only one pipeline
instruments each module.

```ts
import { registerOTel } from '@vercel/otel';
import OpenAI from 'openai';

export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    registerOTel({ serviceName: 'my-service' });

    const { Laminar } = await import('@lmnr-ai/lmnr');
    Laminar.initialize({
      projectApiKey: process.env.LMNR_PROJECT_API_KEY,
      instrumentModules: { openAI: OpenAI },
    });
  }
}
```

## Add manual spans

Use `observe` to wrap the functions that represent meaningful steps.
Pass arguments after the function to capture them as span input (or set `input`
explicitly).

```ts
import { observe } from '@lmnr-ai/lmnr';

const result = await observe(
  {
    name: 'agent.run',
    sessionId: sessionId,
    userId: userId,
    tags: ['agent', 'search'],
    metadata: { route: '/search' },
  },
  async (query, limit) => {
    return await observe({ name: 'retrieve.context' }, async () => {
      // retrieval logic
      return retrieve(query, limit);
    });
  },
  query,
  limit,
);
```

## Privacy and LLM metadata

- Use `ignoreInput`, `ignoreOutput`, or `input` to control what gets captured.
- For manual LLM spans, set `spanType: 'LLM'` and report usage attributes.

```ts
import { Laminar, LaminarAttributes, observe } from '@lmnr-ai/lmnr';

await observe({ name: 'llm.call', spanType: 'LLM', input: { prompt } }, async () => {
  const response = await callModel(prompt);
  Laminar.setSpanAttributes({
    [LaminarAttributes.PROVIDER]: 'openai',
    [LaminarAttributes.REQUEST_MODEL]: response.model,
    [LaminarAttributes.RESPONSE_MODEL]: response.model,
    [LaminarAttributes.INPUT_TOKEN_COUNT]: response.usage?.prompt_tokens,
    [LaminarAttributes.OUTPUT_TOKEN_COUNT]: response.usage?.completion_tokens,
  });
  Laminar.setSpanOutput(response.text);
  return response;
});
```

## Short-lived scripts

Call `await Laminar.flush()` at the end of a script or test to export spans quickly.
