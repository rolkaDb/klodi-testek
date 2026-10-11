# n8n AI workflows

Automation workflows built in [n8n](https://n8n.io/), in the order I learned
them. Everything runs on free services: **Google Gemini** (AI Studio, no card
required) and **Open-Meteo** (no key at all).

One of these is not an exercise: `03-osszetevo-fordito` is the live translation
endpoint behind the [Mentes barcode app](https://github.com/rolkaDb/vonalkod).

## The workflows

| File | What it is | Needs |
|---|---|---|
| `01-idojaras-alapok.json` | n8n fundamentals, no AI | – |
| `02-ai-agent-gemini.json` | An AI agent with tools and memory | Gemini |
| `02-ai-agent-claude.json` | The same with Claude | Anthropic |
| `03-osszetevo-fordito.json` | Webhook serving the Mentes app | Gemini |
| `04a-rag-betoltes.json` | RAG: loading documents | Gemini |
| `04b-rag-chat.json` | RAG: asking the knowledge base | Gemini |

### 01. Weather (fundamentals)

Schedule/Manual trigger → HTTP Request → Code → IF → Set.

No AI in it, and that is deliberate: it teaches n8n's **item model**. Every node
receives and returns a *list* of items. If 10 items go into an HTTP node, it
sends 10 requests. This is n8n's least obvious rule.

### 02. AI Agent

Chat Trigger → AI Agent, with **sub-nodes** hanging below it: the model, memory
and two tools (a calculator and the weather).

Those hanging connectors are not data flow, they are **capabilities**. The agent
decides for itself which tool to call, based on the text of each
`toolDescription`. There is no `if (question contains "weather")` anywhere.

### 03. Ingredient translator

`POST /webhook/osszetevok`, with `{ code, lang, ingredientsText }` in the body
and `{ ok, translatedText, translatedTraces }` in the response.

It serves the [Mentes barcode app](https://github.com/rolkaDb/vonalkod): when an
ingredient text is in a language the app's dictionary does not cover, this
translates it into Hungarian. **The verdict is still produced by the app's
deterministic engine** on the translated text. The AI never decides anything
about an allergen.

Details that matter:

- The validation node rejects empty input and anything over 2,000 characters.
  Anyone can post anything to a webhook.
- In the prompt, the ingredient text sits inside an `<osszetevok>` block, marked
  explicitly as **data, not instructions**. Basic protection against prompt
  injection.
- The model returns "contains" and "may contain traces" on separate lines. Left
  as one block, every trace amount would turn into a block.
- The translation node runs with **Retry On Fail** (3×, 2000 ms) and
  **On Error → Continue**. Without that, a provider's 503 comes back as a 200
  with an empty body.

### 04a / 04b. RAG

Two separate workflows, because RAG is two steps:

```
04a LOADING (once)
  files → chunking → embedding → vector store

04b ASKING (as often as you like)
  question → embedding → similarity search → hits into the prompt → answer
```

The model **does not learn** the documents. On every question we retrieve the
four most similar chunks and hand them to it alongside the question.

Run `04a` first, then the chat in `04b`.

The vector store is **Qdrant**, running as a native binary, so no Docker or WSL
is needed. REST on `6333`, a web interface at `http://localhost:6333/dashboard`.
It survives an n8n restart, which was the only reason for the switch: the earlier
`Simple Vector Store` lived in memory and had to be reloaded every time.

> The first step of `04a` **drops the collection** before reloading. The Qdrant
> node cannot delete, and without this every run would duplicate the documents.
> On the very first run the collection does not exist yet; the node's
> `On Error → Continue` setting swallows that error.

## Getting started

```bash
npx n8n
```

The first run downloads around 1 GB of dependencies; after that it starts in
15 to 20 seconds. The editor is at `http://localhost:5678`. Workflows,
credentials and runs are stored under `~/.n8n/`, so they survive a restart.

> **n8n runs only while the terminal is open.** Close it and the webhooks go
> down with it.

### Importing

> **Every import needs an empty canvas.**
> `Import from File` **adds** the file's contents to the open workflow, it does
> not replace them. Imported onto a canvas that is not empty, the nodes get
> mixed together and the workflow name is overwritten.

The right order: **Overview → Create Workflow → ⋯ → Import from File…**

After importing, the credential has to be **selected by hand** in the model
nodes (`Gemini`, `Embeddings`). It is never part of the exported JSON.

> **Saving is not publishing.** `Ctrl+S` saves a new version, but a live webhook
> keeps using the last published snapshot. After editing, press **Publish**, or
> you get the old behaviour while the database already holds the new code.

### File access for RAG

The file reader node can only read from `~/.n8n-files`, which is n8n 2's default
(`restrictFileAccessTo`). Copy the documents there:

```bash
cp rag-dokumentumok/*.md ~/.n8n-files/rag-dokumentumok/
```

Reading from anywhere else gives "Access to the file is not allowed." You can
widen this with `N8N_RESTRICT_FILE_ACCESS_TO`, but think about what you are
letting in: from then on, any workflow that reaches this n8n can read those
files.

### Gemini key

`aistudio.google.com/apikey` → **Create API key**. No card required. In n8n the
credential is called **Google Gemini(PaLM) Api**.

The free tier has a per-minute request limit that differs by model. An agent
makes **several calls per question**: every tool-calling round is another
request. If you see a lot of `429`s, switch to a Flash or Flash-Lite model.

## rag-dokumentumok/

What `04a` loads, and at the same time the written-up version of what these
workflows taught me:

- `n8n-buktatok.md`: deprecated nodes, `$fromAI`, type-name conventions, with
  the exact error messages
- `gemini-ingyenes.md`: quotas, when you need an agent and when a chain is
  enough
- `vonalkod-app-tervezes.md`: the Mentes app's verdict ranking and source
  marking

---

*A magyar nyelvű változat: [README.hu.md](README.hu.md)*
