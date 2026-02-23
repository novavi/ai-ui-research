# Single Front Door

## High-Level Topics

### 1. Query Handling & Navigation

#### Query Handling

- Distinguishing deterministic (simple, low-cost) queries from non-deterministic (complex, agent-driven) ones
- Routing strategy - rule-based / retrieval-based answers vs full LLM reasoning chains
- Avoiding unnecessary token burn for queries that can be resolved with direct lookup or navigation
- Upsell / cross-sell opportunities e.g. previews of complementary, related products / datasets

#### Navigation

- Prompt-based navigation to complement traditional menu / drill-down UX
- Mapping natural language intent to existing app components / dashboards / reports
- Handling ambiguity - a prompt might map to multiple destinations
- In the short-term, could also support deep-linking into legacy web apps from the chat interface

### 2. Performance

- Users expect visual interfaces to be highly performant when it comes to data selection, filtering, sorting, etc.
- However, when using issuing complex prompts to AI agent chat interfaces with text-based responses, user performance expectations tend to be lower.
- **Significant UX implications with rich visual interactive interfaces in AI platforms. Response latency and interaction model differs markedly between deterministic and non-deterministic paths. There is a risk of creating a mismatch in performance expectations.**

---

### 3. Rendering Strategy: Pre-built vs Generic vs Declarative UI

- **Pre-built renderers**: custom visualisations for specific datasets (e.g. credit risk heatmaps, ownership treemaps) - high fidelity, high maintenance cost
- **Generic renderers**: reusable chart/table components driven by structured data - flexible but lower specificity
- **Declarative / Generative UI**: AI-emitted UI descriptions rendered at runtime (e.g. A2UI patterns) - dynamic but requires guardrails
- Decision framework for choosing the right rendering approach per use case
- Hybrid strategies and fallback chains

---

### 4.1 A2UI Overview

#### Overview

- A2UI (Agent-to-UI) enables Declarative UIs, by allowing AI agents to dynamically compose UIs at runtime
- Instead of relying on a pre-built UI components or emitting raw HTML, agents emit structured JSON schemas describing interface they want rendered, based on the query data shape
- The schema defines component type, layout, data bindings, and interaction handlers - all resolved at render time
- Frontend rendering layer validates schema and maps types to concrete design system components
- Supports streaming - partial schemas can be rendered progressively as the agent response arrives
- Stateless by design - each agent response describes a self-contained UI fragment
- Created by Google with contributions from CopilotKit
- Reached v0.8 as of Feb 2026
- Huge potential, but still very new

#### Why we would use this

- Dynamically compose custom views and dashboards that combine results from multiple datasets
- Agent selects the best representation based on the data shape
- Reduce the need to develop and maintain pre-built components for every possible data combination
- Personalised layouts based on user persona, context or stated preferences

---

### 4.2 A2UI Architecture

```mermaid
flowchart LR
    A([User Prompt]) --> B[Agent / LLM]
    B --> C[UI Schema - JSON Output]
    C --> D{Schema Validator - A2UI}
    D -->|valid| E[Component Resolver - A2UI]
    D -->|invalid| F[Fallback Renderer - A2UI / customisable]
    E --> G[SPGI Design System Components]
    G --> H([Rendered UI Fragment])
    F --> H
    I[(SPGI Component Registry)] -.->|resolves types| E
    C -.->|data extracted from schema| J[(Data / Props)]
    J -.->|injected into| G
```

- Agent never emits raw HTML - all output passes through schema validation before anything renders
- Component Registry is the contract boundary - agents can only reference components that exist in it
- Data and props are embedded directly in the schema by the agent - the agent fetches data first and includes the values inline in the schema output; the rendering layer extracts and injects them into components at render time
- Data and props are injected at render time, keeping the schema itself lightweight
- Fallback renderer handles graceful degradation when schema is malformed or references an unknown component

---

### 4.3 A2UI Example - Rating Card

 Agent composes through component structure - prominent properties are expressed via `CardHeader` (rendered large), secondary properties via `CardBody` fields (rendered small).

```json
{
  "type": "Card",
  "props": {
    "children": [
      {
        "type": "CardHeader",
        "props": {
          "primary": "Apple Inc.",
          "secondary": "AA-"
        }
      },
      {
        "type": "CardBody",
        "props": {
          "fields": [
            { "label": "Rating Date", "value": "15 Feb 2026" },
            { "label": "Prior Rating", "value": "A+" },
            { "label": "Outlook",     "value": "Stable" }
          ]
        }
      }
    ]
  }
}
```

- `Card`, `CardHeader`, and `CardBody` are all registered components in the SPGI Component Registry (agent can only reference component types it knows exist)
- Agent expresses visual hierarchy through component *structure*, not through styling instructions - it has no knowledge of CSS or layout internals
- The schema is compact and data-bearing - `primary`, `secondary`, and `fields` values are all inline data embedded by the agent after fetching the rating from the underlying data API

---

### 4.4 A2UI Implementation Options

| Approach | Who chooses component structure | Flexibility | Reliability |
|---|---|---|---|
| **Fully agent-driven** | The agent | High | Lower - agent can misuse or inconsistently apply components |
| **Template-driven** | Fixed mapping from tool output schema -> component template | Low | High - no ambiguity, consistent output |
| **Hybrid** | Agent picks top-level component; template fills the rest | Medium | Medium |

- For S&P MI, a hybrid or template-driven approach is likely preferable - predictability and consistency matter more than freestyle composition.
- A constrained model where a tool annotates its output with a suggested component, and the agent simply fills in values, is more reliable than asking the agent to compose a ratings card from scratch on every invocation.

##### Considerations

- The agent knows about available components because the A2UI SDK injects the Component Registry into the agent's context. This can be done via system prompt or tool definitions.
- This creates a tension - the larger and more complex the registry, the more context tokens are consumed just describing it.
- The agent needs to understand not just what components *exist* but their *semantic intent* - what `CardHeader.primary` means visually - making component naming and documentation as important as the implementation itself.

---

### 5.1 MCP Apps - Overview

#### Overview

- MCP Apps (official extension to MCP spec) enables cross-organization integration e.g. surfacing S&P MI data in visual form inside partner or customer platforms
- MCP Apps allows MCP servers to return **interactive HTML5 interfaces**
- Whereas plain MCP tools return structured data that an agent / LLM can use, an MCP Apps-enabled tool returns a reference to a self-contained HTML/JS/CSS application that can be rendered by a host
- Created by Anthropic, OpenAI, and the authors of MCP-UI
- Supported in Claude (Anthropic) and ChatGPT (OpenAI) as of Jan 2026
- Reached v1.1 as of Feb 2026
- Generating significant interest across the industry - rich UI responses in AI-based chat products likely to be a key theme in 2026

#### How it works

- Tools declare a `_meta.ui.resourceUri` field, pointing to a self-contained `ui://` resource on the S&P MI MCP server
- When the tool is called, the host (e.g. partner platform) fetches that resource - a bundled HTML+JS+CSS app - and renders it in a **sandboxed iframe**
- Supports bidirectional comms between app and host (JSON-RPC over postMessage)
- Embedded app can request further tool calls, and the host pushes the results back into the app
- Host constructs CSP headers from domains declared by MCP server; undeclared origins are blocked 

---

### 5.2 MCP Apps - Architecture & Data Flow

```mermaid
sequenceDiagram
    participant User
    participant Frontend as Partner Web Frontend
    participant Agent as Partner AI Agent
    participant LLM
    participant Server as SPGI MCP Server
    participant App as SPGI MCP App (iframe)

    User->>Frontend: "show me Apple risk gauge"
    Frontend->>Agent: prompt
    Agent->>LLM: process
    LLM->>Server: tools/call (getRiskGauge)
    Server-->>LLM: data + _meta.ui.resourceUri: ui://risk-gauge-app
    LLM-->>Agent: tool result
    Agent->>Server: resources/read (ui://risk-gauge-app)
    Server-->>Agent: HTML + JS + CSS bundle
    Agent-->>Frontend: HTML content (srcdoc)
    Frontend-->>App: render in sandboxed iframe
    App-->>User: interactive risk dashboard
    User->>App: clicks to drill down on risk drivers
    App->>Frontend: tools/call request (JSON-RPC over postMessage to parent window)
    Frontend->>Agent: proxy tools/call request
    Agent->>Server: tools/call (getRiskDrivers)
    Server-->>Agent: risk drivers data
    Agent-->>Frontend: risk drivers data
    Frontend-->>App: push risk drivers data (JSON-RPC over postMessage to child iframe)
    App-->>User: risk dashboard updates
```

---

### 5.3 MCP Apps - Security

#### Why iframe sandboxing?

- Sandboxed iframes are a standard approach to enforce security in cross-vendor web app integration
- Sandbox guarantees MCP Apps cannot escape their containers or interfere with the host page
- Every app <-> host interaction passes through a logged JSON-RPC channel, making behaviour auditable and restricting injection vectors

#### Authentication

- Handled entirely at the agent-to-MCP-server boundary
- MCP App iframe carries no credentials and requires none
- Data arrives via postMessage from the partner frontend, which proxied it from the agent
- Significant easier to implement than traditional cross-vendor iframe integrations, where the embedded app must authenticate independently (cross-origin OAuth flows, client-side credential management)
