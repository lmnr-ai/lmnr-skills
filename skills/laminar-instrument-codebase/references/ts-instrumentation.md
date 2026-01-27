# TypeScript/JavaScript instrumentation

## Initialize once

Place `Laminar.initialize()` as early as possible (after other instrumentation libraries).

```ts
import { Laminar } from '@lmnr-ai/lmnr';

Laminar.initialize({
  projectApiKey: process.env.LMNR_PROJECT_API_KEY,
  // For self-hosted Laminar:
  // baseUrl: 'http://localhost',
  // httpPort: 8000,
  // grpcPort: 8001,
  // instrumentModules: {}, // disable auto instrumentation
});
```

If you already use another OpenTelemetry setup (for example, Next.js with `@vercel/otel`), use `LaminarSpanProcessor` and `initializeLaminarInstrumentations()` and do NOT call `Laminar.initialize()`.

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

## Add manual spans

Use `observe` to wrap the functions that represent meaningful steps.

```ts
import { observe } from '@lmnr-ai/lmnr';

await observe(
  {
    name: 'agent.run',
    sessionId: sessionId,
    userId: userId,
    tags: ['agent', 'search'],
    metadata: { route: '/search' },
  },
  async () => {
    await observe({ name: 'retrieve.context' }, async () => {
      // retrieval logic
    });
  },
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
