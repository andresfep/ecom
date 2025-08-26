# Veo 3 UGC Video Segment Toolkit

This repository contains a lightweight toolkit for generating detailed, frame-by-frame video description shot lists for UGC-style segments using Google AI models. It includes a server-side service for Gemini/Vertex AI and React components to drive generation from a dashboard.

## What’s in this repo

- `veo3Service.js`: Server-side service that uses either Gemini API (`@google/generative-ai`) or Vertex AI (`@google-cloud/vertexai`) to generate detailed shot-list descriptions for segments.
- `VideoGenerator.js`: React component that submits segments to an API client and displays results with an optional details panel.
- `VideoGeneratorPlus.js`: A streamlined variant of `VideoGenerator` with similar behavior and UI.
- `veo3-json-guidelines.md`: JSON schema and guidance for standard segment objects.
- `veo3-enhanced-continuity.md`: Extended schema with continuity markers for seamless segment transitions.
- `veo3-json-guidelines-plus.md`: Addendum for Plus scenarios.

Note: API clients referenced in components (`../api/client` and `../api/clientPlus`) are not included here. See API setup below.

---

## Prerequisites

- Node.js 18+ (Node 20+ recommended)
- npm or yarn
- One of:
  - Gemini API key (`GOOGLE_GEMINI_API_KEY`), or
  - Vertex AI Project Credentials (Application Default Credentials or service account key)

### Environment Variables

Set one of the following configurations:

- Gemini (hosted API):
  - `GOOGLE_GEMINI_API_KEY`=your_api_key

- Vertex AI (server or Cloud environment):
  - `VERTEX_PROJECT_ID`=your_project_id
  - `VERTEX_LOCATION`=us-central1 (or your region)
  - Optionally rely on `GOOGLE_APPLICATION_CREDENTIALS` (path to service account JSON) or Application Default Credentials.

---

## Installing server-side dependencies

If you plan to run `veo3Service.js` directly:

```bash
npm init -y
npm install @google/generative-ai @google-cloud/vertexai
# (optional) mark ESM if you prefer
npm pkg set type=module
```

> The service file is ESM and uses `import` syntax.

---

## Minimal API server (example)

Create a minimal API to call `veo3Service.js` from your dashboard.

```bash
npm install express cors p-limit
```

Example `server.js`:

```js
import express from 'express';
import cors from 'cors';
import pLimit from 'p-limit';
import veo3Service from './veo3Service.js';

const app = express();
app.use(cors());
app.use(express.json({ limit: '1mb' }));

// limit concurrency to protect rate limits and memory
const limit = pLimit(3);

app.post('/api/generate', async (req, res) => {
  try {
    const { segments, options } = req.body;
    if (!Array.isArray(segments) || segments.length === 0) {
      return res.status(400).json({ error: 'segments[] required' });
    }

    // run with bounded concurrency
    const tasks = segments.map((s, i) =>
      limit(() => veo3Service.generateVideoFromSegment(s, { ...options, segmentIndex: i }))
    );
    const videos = await Promise.all(tasks);
    res.json({ success: true, videos, totalSegments: videos.length });
  } catch (err) {
    res.status(500).json({ error: err.message || 'internal_error' });
  }
});

app.listen(3001, () => console.log('API listening on http://localhost:3001'));
```

---

## Client API modules (examples)

Create these small client wrappers to be used by the React components.

`api/client.js`:

```js
export async function generateVideos(segments, options = {}) {
  const res = await fetch('http://localhost:3001/api/generate', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ segments, options })
  });
  if (!res.ok) throw new Error(`API error ${res.status}`);
  return res.json();
}
```

`api/clientPlus.js`:

```js
export async function generateVideosPlus(segments, options = {}) {
  // Same endpoint for now; you can differentiate server-side via options
  return generateVideos(segments, { ...options, mode: 'plus' });
}
```

---

## Creating a dashboard (quick scaffold)

You can scaffold a lightweight React dashboard with Vite and mount the provided components.

```bash
npm create vite@latest veo3-dashboard -- --template react
cd veo3-dashboard
npm install
mkdir -p src/api
# copy the two client files above into src/api/
# copy VideoGenerator.js and VideoGeneratorPlus.js into src/components/
```

Update `src/main.jsx` and `src/App.jsx` to render a basic dashboard:

`src/App.jsx`:

```jsx
import { useState } from 'react';
import VideoGenerator from './components/VideoGenerator.js';
import VideoGeneratorPlus from './components/VideoGeneratorPlus.js';

export default function App() {
  const [segmentsJson, setSegmentsJson] = useState('[]');
  const [segments, setSegments] = useState([]);

  function loadSegments() {
    try {
      const parsed = JSON.parse(segmentsJson);
      setSegments(parsed);
    } catch (e) {
      alert('Invalid JSON');
    }
  }

  return (
    <div style={{ padding: 20 }}>
      <h2>Veo 3 Dashboard</h2>
      <p>Paste an array of segment objects (see docs in this repo).</p>

      <textarea
        value={segmentsJson}
        onChange={(e) => setSegmentsJson(e.target.value)}
        rows={12}
        style={{ width: '100%' }}
        placeholder="Paste segments JSON here"
      />
      <div style={{ marginTop: 12 }}>
        <button onClick={loadSegments}>Load Segments</button>
      </div>

      {segments.length > 0 && (
        <div style={{ display: 'grid', gridTemplateColumns: '1fr 1fr', gap: 16, marginTop: 20 }}>
          <div style={{ border: '1px solid #ddd', borderRadius: 8, padding: 12 }}>
            <VideoGenerator segments={segments} />
          </div>
          <div style={{ border: '1px solid #ddd', borderRadius: 8, padding: 12 }}>
            <VideoGeneratorPlus segments={segments} />
          </div>
        </div>
      )}
    </div>
  );
}
```

Start the dev server:

```bash
npm run dev
```

Ensure your API server from the previous section is running on `http://localhost:3001`.

---

## Performance guidance (bundle size, load times, optimizations)

### Keep server-only SDKs out of the browser
- Do not import `@google-cloud/vertexai` or `@google/generative-ai` in the client bundle. They are large and require secrets. Use the minimal API wrappers shown above.

### Code-splitting and lazy UI
- Lazy-load details panels to defer large text rendering until requested:
  - Replace the details section with a dynamically imported component (`React.lazy`) or conditional mount to avoid shipping heavy logic by default.
- Only render the expanded `videos` description when the user toggles details.

### Render performance and memory
- Large descriptions can be thousands of characters. Use:
  - Truncated previews with “Show more,” or
  - Virtualized long text views when applicable.
- Avoid storing unnecessary duplicates of `videos` in state. Keep a single source of truth and derive UI from it.

### Network and throughput
- Bound concurrency server-side (see `p-limit(3)` example) to protect rate limits and memory while still parallelizing for responsiveness.
- Consider streaming responses (SSE or chunked transfer) if you plan to show progressive results.

### Tree-shaking and dependencies
- Prefer small helpers over utility megasuites. Do not add client-side SDKs for Google unless strictly needed (server-only recommended).
- Ensure your bundler is ESM-aware and configured for production builds with minification and dead-code elimination.

### Caching
- Cache prompts-to-description results temporarily server-side to avoid repeated charges and latency when users retry.
- If using Vertex AI, reuse model clients rather than re-initializing per request (this repo already reuses a singleton instance via `export default new Veo3Service()`).

### Error handling and resilience
- Surface actionable errors to the UI. Backoff and retry transient errors (429/5xx) with jitter to reduce thundering herds.

---

## Segment JSON format

Refer to:
- `veo3-json-guidelines.md` (standard)
- `veo3-enhanced-continuity.md` (enhanced continuity)
- `veo3-json-guidelines-plus.md` (plus)

Your dashboard should validate these shapes before sending to the API for best results.

---

## Security

- Never expose API keys in the browser. Keep credentials on the server.
- Use CORS restrictions and, in production, authenticated endpoints.

---

## Roadmap

- Streaming token-by-token generation for immediate UI feedback
- Built-in JSON validation and schema hints in the dashboard
- Export-to-JSON and export-to-Markdown for descriptions
- Optional persistence layer for results and caching

---

## License

Proprietary or TBD. Update this section with your team’s chosen license.