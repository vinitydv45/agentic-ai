# Figma API Alternatives Research

## Problem
Figma REST API has strict rate limits (especially for View/Collab seats: ~6 requests/month). This causes 429 errors and blocks project conversions.

## Alternative Approaches

### Option 1: Figma Plugin API (RECOMMENDED)

**How it works:**
- Plugins run **inside** Figma desktop/app
- Direct access to design data (nodes, styles, components) without REST API calls
- Plugin can extract data and send to backend via HTTP/WebSocket

**Pros:**
- ✅ Bypasses REST API rate limits entirely
- ✅ Direct access to design data (faster, more complete)
- ✅ No rate limits for data extraction
- ✅ Can access file data even without API token

**Cons:**
- ❌ Requires users to install plugin
- ❌ Plugin development complexity
- ❌ Users must have Figma desktop app or web app open
- ❌ Plugin execution time limits (~30 seconds)

**Implementation Approach:**
1. Create Figma plugin that:
   - Extracts complete design data (nodes, styles, components, images)
   - Serializes to JSON
   - Sends to backend endpoint via `fetch()` or `XMLHttpRequest`
2. Backend receives data without making REST API calls
3. Fallback to REST API if plugin not available

**Plugin API Documentation:**
- https://www.figma.com/plugin-docs/
- Plugin can access: `figma.root`, `figma.currentPage`, `figma.getNodeById()`
- Can export images: `figma.exportAsync()`

**Next Steps:**
- Create plugin manifest (`manifest.json`)
- Implement plugin UI (optional - can run headless)
- Extract design data using Plugin API
- Send to backend endpoint
- Update backend to accept plugin data

---

### Option 2: Figma MCP Server

**How it works:**
- Figma's Model Context Protocol server for AI agents
- Higher rate limits for Dev/Full seats (similar to Tier 1 REST API)
- Designed specifically for design-to-code workflows

**Pros:**
- ✅ Higher rate limits than REST API for Dev/Full seats
- ✅ Designed for AI agents
- ✅ Can provide design context directly to LLMs
- ✅ Supports both desktop and remote server modes

**Cons:**
- ❌ Still has rate limits (just higher)
- ❌ View/Collab seats still very limited (6 tool calls/month)
- ❌ May not support all operations (images, exports)
- ❌ Requires MCP server setup

**Rate Limits:**
- Dev/Full seats: Similar to REST Tier 1 (~15 requests/min)
- View/Collab seats: 6 tool calls/month (very restrictive)

**Implementation Approach:**
1. Set up Figma MCP server (desktop or remote)
2. Create MCP client wrapper in backend
3. Use MCP for design context extraction
4. Fallback to REST API for images/exports
5. Hybrid approach: MCP for data, REST for assets

**MCP Server Documentation:**
- https://developers.figma.com/docs/figma-mcp-server/
- Remote server: https://developers.figma.com/docs/figma-mcp-server/remote-server-installation/
- Desktop server: https://developers.figma.com/docs/figma-mcp-server/local-server-installation/

**Next Steps:**
- Evaluate MCP server capabilities vs REST API
- Test MCP server integration
- Compare rate limits
- Implement MCP client if viable

---

### Option 3: Webhooks + Incremental Sync

**How it works:**
- Register webhooks for Figma files
- Receive events when files change
- Only fetch changed nodes/pages instead of full file

**Pros:**
- ✅ Drastically reduces API calls (only fetch on changes)
- ✅ Real-time updates
- ✅ Efficient for long-term sync scenarios

**Cons:**
- ❌ Requires webhook infrastructure (public endpoint)
- ❌ May miss events if webhook fails
- ❌ Initial fetch still uses REST API
- ❌ Complex to implement (version tracking, diff logic)

**Implementation Approach:**
1. Register webhooks for files being tracked
2. Receive webhook events (file updates, comments, etc.)
3. Track file versions
4. Fetch only changed nodes/pages
5. Maintain incremental sync state

**Webhook Documentation:**
- https://developers.figma.com/docs/rest-api/webhooks/
- Webhooks V2 API
- Event types: file updates, comments, etc.

**Next Steps:**
- Study webhook API
- Design webhook receiver endpoint
- Plan incremental sync strategy
- Implement if viable

---

## Recommended Strategy

### Phase 1: Plugin API (Primary Solution)
1. **Create Figma Plugin** that extracts design data
2. **Plugin sends data to backend** via HTTP endpoint
3. **Backend processes plugin data** instead of REST API
4. **Fallback to REST API** if plugin not available

### Phase 2: Rate Limit Handling (Fallback)
1. **Keep REST API with retry logic** (already implemented)
2. **Use for images/exports** (plugin may have limitations)
3. **Handle 429 errors gracefully**

### Phase 3: MCP Server (Optional Enhancement)
1. **Evaluate MCP server** if plugin approach has issues
2. **Hybrid approach**: MCP for data, REST for assets
3. **Better rate limits** for Dev/Full seats

---

## Implementation Priority

1. **HIGH**: Plugin API implementation (bypasses limits)
2. **MEDIUM**: Rate limit handling (already done - keep as fallback)
3. **LOW**: MCP Server evaluation (if plugin doesn't work)
4. **LOW**: Webhooks (complex, may not solve immediate problem)

---

## Next Actions

1. ✅ Research Plugin API capabilities
2. ✅ Create plugin prototype (`figma-plugin/`)
3. ✅ Test plugin-to-backend communication
4. ✅ Evaluate MCP Server as alternative
5. ✅ Document findings and recommendations

## Implementation Status

### ✅ Figma Plugin (Option 1) - IMPLEMENTED

Location: `figma-plugin/`

**Features:**
- Extracts complete design data (nodes, styles, colors, fonts, images)
- Exports images as base64
- Sends to backend via HTTP POST
- Configurable backend URL and project settings
- Beautiful dark UI

**Usage:**
1. Build: `cd figma-plugin && npm install && npm run build`
2. Install in Figma Desktop: Menu → Plugins → Development → Import plugin from manifest
3. Run on your design and export to Aura

**Backend Endpoints:**
- `POST /api/figma/plugin-upload` - Dedicated plugin endpoint
- `POST /api/projects/create` with `data_source: "plugin"` - Alternative path

### ✅ Rate Limiting (Fallback) - IMPLEMENTED

Location: `backend/utils/figma_rate_limiter.py`

- Exponential backoff with retry
- Respects Retry-After header
- Request throttling (10 requests/minute)

### ⏳ MCP Server (Option 2) - CONFIGURED

Configuration added but requires MCP server setup.
Set `USE_FIGMA_MCP=true` in `.env` to enable.
