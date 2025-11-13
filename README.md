# Slidev Vue Plugins

A collection of interactive Vue components for [Slidev](https://sli.dev) presentations, designed to enhance educational and technical presentations with live AI interactions and code execution capabilities.

## Components

### 1. LLMQuery - AI-Powered Slide Assistant

An interactive component that integrates multiple Large Language Models (LLMs) directly into your Slidev presentations using [OpenRouter](https://openrouter.ai).

**Features:**
- 🤖 Multi-model support (GPT-4, Claude, Gemini, Llama, Grok, Mistral)
- 💬 Real-time AI responses during presentations
- 🎯 Contextual answers using slide content
- 📍 Flexible positioning (all four corners)
- ⚡ Auto-execute prompts on slide load
- 🎨 Beautiful, responsive UI

**Installation:**

```bash
# Download the component
curl -o components/LLMQuery.vue https://raw.githubusercontent.com/krisluyten/slidev-plugins/main/LLMQuery.vue

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
  model="anthropic/claude-sonnet-4"
  :autoExecute="true"
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
| `model` | `string` | required | LLM model identifier (see supported models) |
| `apiKey` | `string` | env var | OpenRouter API key (optional, uses env) |
| `systemPrompt` | `string` | undefined | Custom system instructions for the AI |
| `prompt` | `string` | undefined | Auto-execute this prompt on slide load |
| `position` | `string` | `'top-right'` | Position: `top-right`, `top-left`, `bottom-right`, `bottom-left` |
| `autoExecute` | `boolean` | `true` | Auto-run prompt when provided |

**Supported Models:**

- `openai/gpt-4.1-mini` - GPT-4.1 Mini
- `anthropic/claude-sonnet-4` - Claude 4 Sonnet
- `anthropic/claude-3.5-sonnet` - Claude 3.5 Sonnet
- `google/gemini-2.5-pro` - Gemini Pro 2.5
- `meta-llama/llama-3.3-70b-instruct` - Llama 3.3 70B
- `x-ai/grok-code-fast-1` - Grok Code Fast
- `x-ai/grok-4` - Grok 4
- `mistralai/mistral-small-3.2-24b-instruct:free` - Mistral Small 3.2

**Examples:**

```vue
<!-- Simple AI assistant -->
<LLMQuery model="anthropic/claude-3.5-sonnet">
  <h1>Machine Learning Basics</h1>
  <p>Introduction to neural networks</p>
</LLMQuery>

<!-- Auto-explaining slide -->
<LLMQuery
  model="openai/gpt-4.1-mini"
  prompt="Summarize the key concepts on this slide"
  :autoExecute="true"
>
  <h1>Complex Topic</h1>
  <ul>
    <li>Point 1</li>
    <li>Point 2</li>
  </ul>
</LLMQuery>

<!-- Code review assistant -->
<LLMQuery
  model="x-ai/grok-code-fast-1"
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
curl -o components/JavaPlayGround.vue https://raw.githubusercontent.com/krisluyten/slidev-plugins/main/JavaPlayGround.vue

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
- GitHub: [@krisluyten](https://github.com/krisluyten)

---

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

---

## Changelog

### 1.0.0 (2025-01-13)
- Initial release
- LLMQuery component with multi-model support
- JavaPlayGround component with dual modes
- Comprehensive documentation and examples
