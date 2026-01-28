# Node/JS quickstart demo (no external LLM calls)

## Steps

1. Create a temp folder and install the SDK:
   - `npm init -y`
   - `npm install @lmnr-ai/lmnr`
2. Export `LMNR_PROJECT_API_KEY`.
3. Create `quickstart.mjs` with the code below.
4. Run `node quickstart.mjs`.
5. Open the Laminar UI -> Traces and filter by tag `quickstart` or metadata `run_id`.

## quickstart.mjs

```js
import { Laminar, observe } from '@lmnr-ai/lmnr';

Laminar.initialize({
  projectApiKey: process.env.LMNR_PROJECT_API_KEY,
  instrumentModules: {},
  disableBatch: true,
  // For self-hosted Laminar:
  // baseUrl: 'http://localhost',
  // httpPort: 8000,
  // grpcPort: 8001,
});

const runId = `quickstart-${Date.now()}`;

const result = await observe(
  { name: 'quickstart.root', tags: ['quickstart', runId], metadata: { run_id: runId } },
  async () => {
    const step = await observe({ name: 'quickstart.step' }, async () => {
      return { answer: 42 };
    });
    return { step };
  },
);

console.log('Laminar quickstart run id:', runId);
console.log('Result:', result);
await Laminar.flush(); // IMPORTANT: await is crucial, and this line is required in short-lived scripts
```

## Notes

- Use a `.mjs` file or set `"type": "module"` in `package.json` for ESM imports.
- If you prefer batching, remove `disableBatch: true` and keep `Laminar.flush()`.
