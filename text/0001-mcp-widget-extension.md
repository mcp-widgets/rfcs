
# Model Context Protocol (MCP) Widget Extension — Draft Specification v0.1

**Author:** Nicklas Scharpff
**Created:** 2025-04-03
**Status:** Draft / Open for Feedback  
**Repo:** https://github.com/mcp-widgets/rfcs

---

## Overview

This extension introduces an optional `widget` field in MCP tool response payloads, enabling AI platforms to render **branded, interactive, and structured frontend components** (Model Context Widgets / MCWs) alongside traditional text responses.

The goal is to bridge structured backend data with **actionable UI elements**, without compromising the core functionality of existing MCP tools.

---

## Goals

- Maintain full **backward compatibility** with MCP
- Enable **rich UI rendering** inside AI interfaces (chat, voice, agent UIs)
- Preserve **brand ownership and interactivity** for the service provider
- Support **secure sandboxing** and graceful fallbacks

---

## Example Response (Extended MCP)

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
    "fallback_text": "Ordering an iPhone 15 for $999. Please confirm."
  }
}
```

---

## `widget` Object Specification

### Top-Level Fields

| Field           | Type                                   | Required | Description                                                                 |
|----------------|----------------------------------------|----------|-----------------------------------------------------------------------------|
| `type`         | `"iframe"` \| `"html_embed"` \| `"web_component"` | Yes        | How the widget should be rendered                                           |
| `fallback_text`| `string`                                | Yes        | What to show if widget rendering is not supported                          |

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

## Security Considerations

- **Sandboxing is strongly recommended** for all rendering modes.
- Widgets should not assume access to global DOM or parent app styles.
- Clients should validate widget source domains and enforce CSP headers.

---

## Design Considerations

- Platforms may wrap widgets in a container for UX consistency.
- Developers should support theming and accessibility where possible.
- Fallback text is required for non-visual UIs and accessibility.

---

## Sample Use Cases

- E-commerce checkout with brand UI (e.g., Apple, Amazon)
- Ticketing or task management (Jira, Notion)
- Travel booking confirmations
- Dashboard mini-widgets (e.g., calendar, finance)
- Approve/sign flows for documents or transactions

---

## Real-World Implementation Notes

In early implementations, I observed the need for practical compatibility with the current MCP spec. Since `widget` is not yet a recognized top-level `type`, I used:

```json
{
  "type": "resource",
  "resource": {
    "text": "Forecast preview",
    "uri": "data:text/html,...",
    "mimeType": "text/html"
  }
}
```

The URI included a server-side rendered React component, which was purified and inserted on the client using React's `dangerouslySetInnerHTML` in the model's response message.
This approach allowed the owner of the MCP server to fully control the UI rendered alongside the MCP data response.


This is how I implemented it in my example project. It's quite a bit of boilerplate, which could be extracted into a separate package.
```tsx
 
const RenderToolWidget = ({ result }: { result: result }) => {
  // Extract the resource from potentially nested structure
  let resource: any;

  if (result?.content && Array.isArray(result.content)) {
    // Handle case where result has a content array (direct MCP response)
    const resourceItem = result.content.find(
      (item: any) => item.type === 'resource',
    );
    resource = resourceItem?.resource;
  } else {
    // Assume result is the resource itself
    resource = result;
  }

  // For data URI containing HTML
  if (resource?.uri?.startsWith('data:text/html')) {
    try {
      const htmlContent = decodeURIComponent(
        resource.uri.replace('data:text/html,', ''),
      );
      
      // Configure DOMPurify to keep all classes and styles
      const sanitizeConfig = {
        ADD_ATTR: ['class', 'style'],
        ADD_TAGS: ['style'],
        KEEP_CONTENT: true,
        USE_PROFILES: { html: true },
      };
      
      // Create a container element to isolate the widget styles
      const containerId = `mcp-widget-${Math.random().toString(36).substring(2, 9)}`;
      
      // Extract styles from the HTML and make them scoped to our container
      let styleContent = '';
      const styleMatch = htmlContent.match(/<style[^>]*>([\s\S]*?)<\/style>/i);
      if (styleMatch?.[1]) {
        styleContent = styleMatch[1];
      }
      
      // Extract the body content
      let bodyContent = '';
      const bodyMatch = htmlContent.match(/<body[^>]*>([\s\S]*?)<\/body>/i);
      if (bodyMatch?.[1]) {
        bodyContent = bodyMatch[1];
      } else {
        // If no body tags, assume the whole HTML is the content we want
        // but remove any style tags first
        bodyContent = htmlContent.replace(/<style[^>]*>[\s\S]*?<\/style>/gi, '');
      }

      const sanitizedHtml = DOMPurify.sanitize(bodyContent, sanitizeConfig);

      return (
        <div 
          id={containerId}
          className="mcp-widget-container w-full overflow-hidden rounded-xl relative"
          data-has-html-content="true"
        >
          <style dangerouslySetInnerHTML={{ __html: styleContent }} />
          <div dangerouslySetInnerHTML={{ __html: sanitizedHtml }} />
        </div>
      );
    } catch (e) {
      console.error('Error parsing HTML content', e);
    }
  }
```
---

## Interim: Embedding Widgets in `resource` Blocks

Until `type: "widget"` is accepted in the MCP spec, platforms may embed widget metadata within a `resource` block:

```json
{
  "type": "resource",
  "resource": {
    "text": "Forecast for San Francisco",
    "uri": "data:text/html,...",
    "mimeType": "text/html",
    "x-widget": {
      "type": "html_embed",
      "fallback_text": "72°F and sunny in SF",
      "props": {
        "latitude": 37.77,
        "longitude": -122.42
      },
      "bundle_url": "https://example.com/widgets/weather.bundle.js"
    }
  }
}
```

--

## Versioning

This spec represents **v0.1** of the `widget` extension to MCP.  
It is expected to evolve based on community feedback and adoption.

---

## Call for Feedback

Want to test, implement, or improve this spec?

- Open an issue or discussion: [GitHub link]
- Try the demo: [demo link]
- Share your ideas or implementations!

---

## 🔗 Related Resources

- [Anthropic Model Context Protocol (GitHub)](https://github.com/anthropics/model-context-protocol)
- [This Spec GitHub Repo](https://github.com/mcp-widgets/rfcs)
- [MCW Demo Site](https://github.com/mcp-widgets/examples)

