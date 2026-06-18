# Figma MCP Server Setup Guide

## Overview

The Figma MCP Server provides an alternative way to fetch design data that bypasses REST API rate limits. It's designed specifically for AI agents and provides higher rate limits for Dev/Full seats.

## Benefits

- **Higher Rate Limits**: Dev/Full seats get ~15 requests/min (similar to REST Tier 1)
- **Designed for AI Agents**: Optimized for design-to-code workflows
- **Reduced API Calls**: Can extract design context more efficiently
- **Better Context**: Provides richer design context for AI agents

## Setup Options

### Option 1: Remote MCP Server (Recommended)

The remote MCP server is hosted by Figma and doesn't require the desktop app.

**Requirements:**
- Figma account with Dev or Full seat
- OAuth authentication setup

**Configuration:**
1. Enable in `.env`:
   ```bash
   USE_FIGMA_MCP=true
   FIGMA_MCP_SERVER_TYPE=remote
   FIGMA_MCP_SERVER_URL=https://mcp.figma.com/mcp
   ```

2. Complete OAuth flow (first time only):
   - The MCP client will prompt for OAuth authentication
   - Follow the authentication flow in your browser
   - Token will be stored for future use

**Documentation:**
- https://developers.figma.com/docs/figma-mcp-server/remote-server-installation/

### Option 2: Local/Desktop MCP Server

Runs via the Figma desktop app on your local machine.

**Requirements:**
- Figma desktop app installed
- MCP server plugin/extension

**Configuration:**
1. Install Figma desktop app
2. Set up MCP server locally
3. Enable in `.env`:
   ```bash
   USE_FIGMA_MCP=true
   FIGMA_MCP_SERVER_TYPE=local
   ```

**Documentation:**
- https://developers.figma.com/docs/figma-mcp-server/local-server-installation/

## Current Implementation

The system is configured to:
1. **Use REST API by default** (with rate limiting and retry logic)
2. **Optionally use MCP Server** if enabled in settings
3. **Fallback to REST API** if MCP is unavailable or fails

## Rate Limits Comparison

| Method | View/Collab Seats | Dev/Full Seats |
|--------|------------------|----------------|
| REST API | ~6 calls/month | ~15-20 calls/min |
| MCP Server | ~6 tool calls/month | ~15 tool calls/min |

**Note**: MCP Server has similar limits but is designed for AI agents and may be more efficient.

## Testing

To test the MCP Server integration:

1. **Enable MCP Server**:
   ```bash
   # In .env file
   USE_FIGMA_MCP=true
   FIGMA_MCP_SERVER_TYPE=remote
   ```

2. **Start the server**:
   ```bash
   python -m uvicorn backend.main:app --reload
   ```

3. **Create a test project**:
   ```bash
   curl -X POST http://localhost:8000/api/projects/create \
     -H "Content-Type: application/json" \
     -d '{
       "figma_url": "https://www.figma.com/design/YOUR_FILE_KEY/Your-Design",
       "project_name": "test-mcp",
       "ui_library": "tailwind"
     }'
   ```

4. **Check logs** for MCP usage:
   - Look for `[MCP] Figma MCP Server enabled` message
   - Monitor for any MCP tool calls in agent logs

## Troubleshooting

### MCP Server Not Connecting

- **Check OAuth**: Ensure OAuth flow completed successfully
- **Check URL**: Verify `FIGMA_MCP_SERVER_URL` is correct
- **Check Seat Type**: Ensure account has Dev or Full seat
- **Fallback**: System will automatically fallback to REST API

### Rate Limits Still Hit

- **Check Seat Type**: MCP has same limits for View/Collab seats
- **Upgrade Seat**: Consider upgrading to Dev/Full seat
- **Use REST API**: System will use REST API with rate limiting as fallback

## Next Steps

1. ✅ MCP Server configuration added
2. ⏳ Test with remote MCP server
3. ⏳ Verify rate limit improvements
4. ⏳ Document any issues or limitations

## References

- [Figma MCP Server Documentation](https://developers.figma.com/docs/figma-mcp-server/)
- [Remote Server Setup](https://developers.figma.com/docs/figma-mcp-server/remote-server-installation/)
- [Local Server Setup](https://developers.figma.com/docs/figma-mcp-server/local-server-installation/)
- [Plans and Permissions](https://developers.figma.com/docs/figma-mcp-server/plans-access-and-permissions/)
