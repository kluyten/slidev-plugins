# Slidev Vue Plugins

A collection of interactive Vue components for [Slidev](https://sli.dev) presentations, designed to enhance educational and technical presentations with live AI interactions and code execution capabilities.

## Components

### 1. LLMQuery - AI-Powered Slide Assistant

An interactive component that integrates multiple Large Language Models (LLMs) directly into your Slidev presentations using [OpenRouter](https://openrouter.ai).

**Features:**
- 🤖 Multi-model support through OpenRouter (Claude, GPT, Gemini, Llama, Grok, Mistral, DeepSeek, Kimi, Qwen)
- 🔀 Model picker in the panel, so you can switch models during a talk
- 💬 Real-time AI responses during presentations
- 🎯 Contextual answers using slide content
- 📍 Flexible positioning (all four corners)
- ⚡ Prompts are fetched when the slide loads, so the answer is ready when you click
- 🖨️ No API calls during PDF/PNG export

**Installation:**

```bash
# Download the component
curl -o components/LLMQuery.vue https://raw.githubusercontent.com/kluyten/slidev-plugins/main/LLMQuery.vue

# Install dependencies
npm install axios marked
```

**Setup:**

Create a `.env` file in your Slidev project root:

```env
VITE_OPENROUTER_API_KEY=your_openrouter_api_key
```

Get your free API key at [OpenRouter](https://openrouter.ai/keys)

**Usage:**

```vue
<LLMQuery
  model="anthropic/claude-haiku-4.5"
  prompt="Explain this slide in simple terms"
  position="top-right"
>
  <h1>Your Slide Content</h1>
  <p>The AI will use this content as context</p>
</LLMQuery>
```

**Props:**

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `model` | `string` | `anthropic/claude-haiku-4.5` | OpenRouter model id the panel starts on (see models below) |
| `apiKey` | `string` | env var | OpenRouter API key (optional, uses env) |
| `systemPrompt` | `string` | undefined | Custom system instructions for the AI |
| `prompt` | `string` | undefined | The question. If set, it is fetched when the slide loads |
| `position` | `string` | `'top-right'` | Position: `top-right`, `top-left`, `bottom-right`, `bottom-left` |
| `autoExecute` | `boolean` | `true` | Fetch `prompt` when the slide loads. `:autoExecute="false"` fetches only on click |

**How it works:**

1. The slot content is rendered on the slide and its text is appended to `prompt`, so the model
   sees the slide it is being asked about.
2. The request goes to OpenRouter with `VITE_OPENROUTER_API_KEY` from `.env`.
3. The reply is rendered as Markdown with `marked`.

**Auto-execute fetches the answer but does not open the panel.** A panel that opens by itself would
cover the slide. A badge on the button shows the state of the fetched answer:

| Badge | Meaning |
|---|---|
| `Auto` (grey) | not fetched yet, or `:autoExecute="false"` |
| `...` (amber, pulsing) | the model is still answering |
| `✓` (green) | the answer is ready |
| `!` (red) | the call failed; click to see the error |

**Exports do not call the API.** Slidev's PDF/PNG export loads every slide in a headless browser.
The component detects export mode and skips the fetch, so an export costs nothing.

**Models:**

The panel has a dropdown to switch models. A choice applies to that panel only and is reset on
reload. `modelConfigs` in `LLMQuery.vue` lists the known models; prices are USD per million tokens
(input / output) at the time of writing.

| Group | Model id | Shown as |
|---|---|---|
| Fast | `anthropic/claude-haiku-4.5` | Claude Haiku 4.5 |
| Fast | `openai/gpt-5.6-luna` | GPT-5.6 Luna |
| Fast | `google/gemini-3.1-flash-lite` | Gemini 3.1 Flash Lite |
| Fast | `meta-llama/llama-4-scout` | Llama 4 Scout |
| Fast | `mistralai/ministral-8b-2512` | Ministral 8B |
| General | `anthropic/claude-sonnet-5` | Claude Sonnet 5 |
| General | `google/gemini-3.6-flash` | Gemini 3.6 Flash |
| General | `openai/gpt-4.1` | GPT-4.1 |
| General | `x-ai/grok-4.3` | Grok 4.3 |
| General | `deepseek/deepseek-v4-pro-0813` | DeepSeek V4 Pro |
| Reasoning | `anthropic/claude-opus-5` | Claude Opus 5 |
| Reasoning | `openai/gpt-5.6-terra-pro` | GPT-5.6 Terra Pro |
| Reasoning | `x-ai/grok-4.6` | Grok 4.6 |
| Reasoning | `moonshotai/kimi-k3` | Kimi K3 |
| Coding | `openai/gpt-5.3-codex` | GPT-5.3 Codex |
| Coding | `qwen/qwen3-coder-flash` | Qwen3 Coder Flash |
| Coding | `mistralai/devstral-2512` | Devstral 2512 |

Any other OpenRouter id also works; it is shown with a question-mark icon and logs a console
warning, so a typo or a retired model is easy to spot. Ids from version 1.0.0 that OpenRouter has
retired (`x-ai/grok-code-fast-1`, `x-ai/grok-4`, `anthropic/claude-3.5-sonnet`,
`anthropic/claude-sonnet-4`, `google/gemini-2.5-pro`, `gpt-4`,
`mistralai/mistral-small-3.2-24b-instruct:free`) are mapped to a current replacement, so older
decks keep working.

> **Model ids go out of date.** OpenRouter retires ids without notice. Check the list before you
> present:
> ```bash
> curl -s https://openrouter.ai/api/v1/models | \
>   python3 -c "import json,sys; print('\n'.join(sorted(m['id'] for m in json.load(sys.stdin)['data'])))"
> ```

**Examples:**

```vue
<!-- Simple AI assistant, default model -->
<LLMQuery>
  <h1>Machine Learning Basics</h1>
  <p>Introduction to neural networks</p>
</LLMQuery>

<!-- Answer fetched when the slide loads -->
<LLMQuery
  model="google/gemini-3.1-flash-lite"
  prompt="Summarize the key concepts on this slide"
>
  <h1>Complex Topic</h1>
  <ul>
    <li>Point 1</li>
    <li>Point 2</li>
  </ul>
</LLMQuery>

<!-- Code review assistant -->
<LLMQuery
  model="qwen/qwen3-coder-flash"
  systemPrompt="You are a code review expert. Focus on bugs and best practices."
  position="bottom-right"
>
  ```python
  def calculate(x):
      return x * 2
  ```
</LLMQuery>
```

**Blog Post:** [LLMQuery for Slidev - Full Tutorial](https://krisluyten.net/news/2025/07/29/llmquery_for_slidev/)

---

### 2. JavaPlayGround - Interactive Java Code Editor

A live Java code editor component with real-time execution capabilities, perfect for coding demonstrations and educational presentations.

**Features:**
- ☕ Live Java code editing and execution
- 🎯 Two modes: iframe embed (no setup) or API mode
- 🎨 Syntax-aware textarea with proper formatting
- 📊 Optional execution statistics
- 🔄 Reset functionality
- 📱 Responsive and customizable layout

**Installation:**

```bash
# Download the component
curl -o components/JavaPlayGround.vue https://raw.githubusercontent.com/kluyten/slidev-plugins/main/JavaPlayGround.vue

# Install dependencies
npm install axios
```

**Setup (API Mode):**

Create a `.env` file for JDoodle API access:

```env
VITE_JDOODLE_CLIENT_ID=your_client_id
VITE_JDOODLE_CLIENT_SECRET=your_client_secret
```

Get free API keys at [JDoodle Compiler API](https://www.jdoodle.com/compiler-api) (200 requests/day free tier)

**Vite Proxy Setup (Development):**

Add to your `vite.config.ts` to bypass CORS in development:

```typescript
export default defineConfig({
  server: {
    proxy: {
      '/api/jdoodle': {
        target: 'https://api.jdoodle.com',
        changeOrigin: true,
        rewrite: (path) => path.replace(/^\/api\/jdoodle/, ''),
      },
    },
  },
})
```

**Usage:**

**Mode 1: Iframe Embed (Recommended for presentations)**

```vue
<JavaPlayGround
  embedUrl="https://www.jdoodle.com/embed/v0/8gHd"
  editorHeight="500px"
/>
```

**Mode 2: Built-in Editor with API**

```vue
<script setup>
const code = `public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}`
</script>

<JavaPlayGround
  :code="code"
  fileName="HelloWorld.java"
  editorHeight="400px"
  :showStats="true"
/>
```

**Props:**

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `embedUrl` | `string` | undefined | JDoodle embed URL (enables iframe mode) |
| `code` | `string` | template | Initial Java code (editor mode) |
| `fileName` | `string` | undefined | Display file name in header |
| `editorHeight` | `string` | `'500px'` | Height of the editor/iframe |
| `compact` | `boolean` | `false` | Compact mode with smaller fonts |
| `showOutput` | `boolean` | `true` | Show output panel (editor mode) |
| `showStats` | `boolean` | `false` | Show execution time/memory stats |

**Examples:**

```vue
<!-- Simple example with custom code -->
<script setup>
const threadDemo = `public class SimpleThread extends Thread {
  private int countDown = 5;

  public void run() {
    while(countDown-- > 0) {
      System.out.println("Thread: " + countDown);
    }
  }

  public static void main(String[] args) {
    new SimpleThread().start();
  }
}`
</script>

<JavaPlayGround
  :code="threadDemo"
  fileName="SimpleThread.java"
  editorHeight="420px"
/>

<!-- Embedded JDoodle (no setup required) -->
<JavaPlayGround
  embedUrl="https://www.jdoodle.com/embed/v0/8gHd"
  editorHeight="600px"
/>

<!-- Compact mode with stats -->
<JavaPlayGround
  :code="code"
  :compact="true"
  :showStats="true"
  editorHeight="350px"
/>
```

**Creating JDoodle Embeds:**

1. Go to [JDoodle](https://www.jdoodle.com/online-java-compiler)
2. Write your code
3. Click "Save" → Get the embed URL
4. Use the embed URL in the `embedUrl` prop

---

## License

MIT License - Copyright (c) 2025 Kris Luyten

See individual component files for full license text.

---

## Author

**Kris Luyten**
- Website: [https://krisluyten.net](https://krisluyten.net)
- GitHub: [@kluyten](https://github.com/kluyten)

---

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

---

## Changelog

### 1.1.0 (2026-10-18)
- LLMQuery: model picker in the panel, grouped into fast, general, reasoning and coding models
- LLMQuery: `model` is optional and defaults to Claude Haiku 4.5
- LLMQuery: model list updated to current OpenRouter ids; retired ids map to a replacement
- LLMQuery: unknown model ids show a question-mark icon and a console warning
- LLMQuery: auto-execute fetches the answer without opening the panel; a badge shows its state
- LLMQuery: fix auto-execute never running when the prop was omitted
- LLMQuery: no API calls during PDF/PNG export

### 1.0.0 (2025-01-13)
- Initial release
- LLMQuery component with multi-model support
- JavaPlayGround component with dual modes
- Comprehensive documentation and examples
