# Technology Intelligence Studio — v6

A standalone Persian RTL technology intelligence dashboard for AvalAI model discovery, architecture advice, free/open-source agent discovery, Python/JavaScript libraries, media workflows and social publishing.

## Run

Open `index.html` through GitHub Pages or any static HTTPS host. Some browsers restrict `fetch()` from `file://`, so a local HTTP server or GitHub Pages is recommended.

## v6 additions

- Free/open-source agent marketplace with categories: research, coding, browser, multi-agent, content and social.
- Live GitHub repository search for agent projects (public unauthenticated API; rate limits apply).
- Technology library covering agent frameworks, memory/RAG, content, media, social APIs, web research and workflow/queues.
- Runtime fit labels distinguish Edge/Worker-friendly, Python runtime and server-heavy components.
- Rules-based project-to-stack planner, with option to send the project brief to the AvalAI advisor.
- Existing v5 model catalogue, advisor, cost tools, GitHub health check, architecture and revenue sections retained.

## Important limitations

- GitHub search results are discovery leads, not endorsements. Check repository activity, license, releases, dependencies, security and commercial-use rights before adoption.
- “Free/open source” does not mean free operation: model API, search API, hosting, GPU, storage and social platform access may cost money.
- No VPS is assumed. Cloudflare Workers are for API/webhook/short orchestration jobs, not FFmpeg, Playwright browser binaries, Whisper local inference or long-running Python workers.
- Python Hosting capabilities (background process, FFmpeg, disk, RAM, execution limits) must be verified before selecting media tools.
- Social publishing should use official APIs and proper OAuth/scopes. Browser automation is brittle and must not bypass CAPTCHAs, access controls or platform rules. Keep human approval for publishing.
- AvalAI API keys typed in the browser are held only in page memory in this static prototype; do not deploy this as a public multi-user app. Move secrets and provider calls to a secured Worker/backend before public release.
- Model metadata and inferred recommendations are not official quality benchmarks. Validate exact endpoint compatibility and current pricing against AvalAI documentation.
