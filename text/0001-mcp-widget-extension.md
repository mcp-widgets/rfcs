
# 🧩 Model Context Protocol (MCP) Widget Extension — Draft Specification v0.1

**Author:** Nicklas Scharpff
**Created:** 2025-04-03
**Status:** Draft / Open for Feedback  
**Repo:** https://github.com/mcp-widgets/rfcs

---

## 📌 Overview

This extension introduces an optional `widget` field in MCP tool response payloads, enabling AI platforms to render **branded, interactive, and structured frontend components** (Model Context Widgets / MCWs) alongside traditional text responses.

The goal is to bridge structured backend data with **actionable UI elements**, without compromising the core functionality of existing MCP tools.

---

## ✅ Goals

- Maintain full **backward compatibility** with MCP
- Enable **rich UI rendering** inside AI interfaces (chat, voice, agent UIs)
- Preserve **brand ownership and interactivity** for the service provider
- Support **secure sandboxing** and graceful fallbacks

---

## 🧱 Example Response (Extended MCP)

```json
{
  "type": "tool_use_result",
  "name": "apple.checkout",
  "content": {
    "item": "iPhone 15",
    "price": 999
  },
  "widget": {
    "type": "iframe",
    "url": "https://apple.com/widgets/checkout?session=abc123",
    "sandbox": true,
    "fallback_text": "Ordering an iPhone 15 for $999. Click here to confirm."
  }
}
```

---

## 🧰 `widget` Object Specification

### Top-Level Fields

| Field           | Type                                   | Required | Description                                                                 |
|----------------|----------------------------------------|----------|-----------------------------------------------------------------------------|
| `type`         | `"iframe"` \| `"html_embed"` \| `"web_component"` | ✅        | How the widget should be rendered                                           |
| `fallback_text`| `string`                                | ✅        | What to show if widget rendering is not supported                          |

---

### Type: `iframe`

```json
"widget": {
  "type": "iframe",
  "url": "https://example.com/widget",
  "sandbox": true,
  "width": "400px",
  "height": "auto",
  "theme": "auto"
}
```

| Field    | Type     | Description                                      |
|----------|----------|--------------------------------------------------|
| `url`    | `string` | URL to fetch the widget UI                       |
| `sandbox`| `boolean`| Should be rendered in a sandboxed iframe         |
| `width`  | `string` | Optional CSS width (e.g., "100%" or "400px")     |
| `height` | `string` | Optional CSS height                              |
| `theme`  | `string` | Optional hint: `"light"`, `"dark"`, `"auto"`     |

---

### Type: `html_embed`

```json
"widget": {
  "type": "html_embed",
  "html": "<div class='my-widget'>...</div>",
  "css": ".my-widget { color: red; }",
  "js": "<script>...</script>",
  "sandbox": true
}
```

| Field  | Type     | Description                                       |
|--------|----------|---------------------------------------------------|
| `html` | `string` | Raw HTML to render                                |
| `css`  | `string` | Scoped CSS for styles                             |
| `js`   | `string` | Optional embedded JavaScript                      |
| `sandbox` | `boolean` | Should be sandboxed (e.g., via shadow DOM)     |

---

### Type: `web_component`

```json
"widget": {
  "type": "web_component",
  "tag_name": "custom-checkout",
  "props": {
    "product": "iPhone 15",
    "price": "$999"
  },
  "bundle_url": "https://example.com/bundles/checkout.js"
}
```

| Field       | Type     | Description                                     |
|-------------|----------|-------------------------------------------------|
| `tag_name`  | `string` | Custom element tag name                         |
| `props`     | `object` | JSON props passed to the element                |
| `bundle_url`| `string` | JavaScript bundle to register the component     |

---

## 🔄 Optional Interactions

```json
"interactions": {
  "confirmOrder": {
    "type": "postMessage",
    "target_action": "apple.confirm_order"
  }
}
```

- Enables widgets to communicate actions back to the agent or AI client.
- Future support: `callback_url`, `eventBridge`, `webhooks`.

---

## 🛡 Security Considerations

- **Sandboxing is strongly recommended** for all rendering modes.
- Widgets should not assume access to global DOM or parent app styles.
- Clients should validate widget source domains and enforce CSP headers.

---

## 🧠 Design Considerations

- Platforms may wrap widgets in a container for UX consistency.
- Developers should support theming and accessibility where possible.
- Fallback text is required for non-visual UIs and accessibility.

---

## 🧪 Sample Use Cases

- E-commerce checkout with brand UI (e.g., Apple, Amazon)
- Ticketing or task management (Jira, Notion)
- Travel booking confirmations
- Dashboard mini-widgets (e.g., calendar, finance)
- Approve/sign flows for documents or transactions

---

## 📅 Versioning

This spec represents **v0.1** of the `widget` extension to MCP.  
It is expected to evolve based on community feedback and adoption.

---

## 🙋 Call for Feedback

Want to test, implement, or improve this spec?

- Open an issue or discussion: [GitHub link]
- Try the demo: [demo link]
- Share your ideas or implementations!

---

## 🔗 Related Resources

- [Anthropic Model Context Protocol (GitHub)](https://github.com/anthropics/model-context-protocol)
- [This Spec GitHub Repo](https://github.com/mcp-widgets/rfcs)
- [MCW Demo Site](#)

