# Aura2 - Figma-to-React API Documentation

**Base URL:** `http://localhost:8000`
**API Version:** 1.0.0
**Last Updated:** 2026-01-18

This document describes all available API endpoints for the Aura2 Figma-to-React conversion platform. Use these endpoints to build your frontend application.

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication](#authentication)
3. [API Endpoints](#api-endpoints)
   - [Health Check](#health-check)
   - [Create Project](#create-project)
   - [Add Website](#add-website)
   - [Get Project Status](#get-project-status)
   - [List Projects](#list-projects)
   - [Delete Project](#delete-project)
   - [List Components](#list-components)
   - [Get Statistics](#get-statistics)
4. [Data Models](#data-models)
5. [Error Handling](#error-handling)
6. [Examples](#examples)

---

## Overview

The Aura2 API provides endpoints to:
- Create React projects from Figma designs
- Track conversion progress
- Manage project library
- View component reuse statistics

All endpoints return JSON responses and use standard HTTP status codes.

---

## Authentication

Currently, the API does not require authentication. Figma token is configured server-side in the `.env` file.

**Future Enhancement:** API keys for multi-user support.

---

## API Endpoints

### Health Check

Get API information and verify the server is running.

**Endpoint:** `GET /`

**Response:**
```json
{
  "name": "Aura2 - Figma-to-React Generator",
  "version": "1.0.0",
  "docs": "/docs"
}
```

**Status Codes:**
- `200 OK` - Server is running

---

### Create Project

Create a new React project from a Figma design. This is the **first project** - components are created from scratch.

**Endpoint:** `POST /api/projects/create`

**Request Body:**
```json
{
  "figma_url": "string (required)",
  "project_name": "string (required)",
  "ui_library": "string (optional, default: tailwind)"
}
```

**Parameters:**
- `figma_url`: Full Figma file URL (e.g., `https://www.figma.com/design/xxx/...`)
- `project_name`: Unique project name (alphanumeric, hyphens, underscores)
- `ui_library`: UI framework - Options: `"tailwind"`, `"mui"`, `"chakra"`

**Response:** `200 OK`
```json
{
  "project_id": "7",
  "status": "pending",
  "message": "Project creation started with TAILWIND. First project - all components will be created from scratch and saved to library."
}
```

**Status Codes:**
- `200 OK` - Project creation started (background task)
- `400 Bad Request` - Invalid parameters or project name already exists

**Example:**
```bash
curl -X POST http://localhost:8000/api/projects/create \
  -H "Content-Type: application/json" \
  -d '{
    "figma_url": "https://www.figma.com/design/a577puJyvBsiQljgPCh4IA/Samsung-Website",
    "project_name": "my-awesome-website",
    "ui_library": "tailwind"
  }'
```

**Notes:**
- Conversion runs in background (async)
- Use [Get Project Status](#get-project-status) to track progress
- Estimated time: 5-10 minutes depending on design complexity

---

### Add Website

Add another website to the platform. This endpoint **maximizes component reuse** from existing library.

**Endpoint:** `POST /api/projects/add-website`

**Request Body:**
```json
{
  "figma_url": "string (required)",
  "project_name": "string (required)",
  "ui_library": "string (optional, default: tailwind)"
}
```

**Parameters:** Same as [Create Project](#create-project)

**Response:** `200 OK`
```json
{
  "project_id": "8",
  "status": "pending",
  "message": "Website conversion started with TAILWIND. Analyzing component library for reuse opportunities..."
}
```

**Status Codes:**
- `200 OK` - Conversion started
- `400 Bad Request` - Invalid parameters

**Example:**
```bash
curl -X POST http://localhost:8000/api/projects/add-website \
  -H "Content-Type: application/json" \
  -d '{
    "figma_url": "https://www.figma.com/design/yyy/Another-Design",
    "project_name": "another-website",
    "ui_library": "tailwind"
  }'
```

**Difference from Create Project:**
- Always enables component reuse analysis
- Agent searches library for similar components (>80% similarity)
- Faster conversion if many components can be reused

---

### Get Project Status

Get the current status and details of a project.

**Endpoint:** `GET /api/projects/{project_id}/status`

**Path Parameters:**
- `project_id`: Integer - Project ID returned from create/add endpoints

**Response:** `200 OK`
```json
{
  "id": 7,
  "name": "samsung-test-enhanced",
  "status": "success",
  "figma_url": "https://www.figma.com/design/a577puJyvBsiQljgPCh4IA/...",
  "project_path": "generated_projects/samsung-test-enhanced",
  "components_generated": 6,
  "components_reused": 0,
  "conversion_time_seconds": 432.39,
  "created_at": "2026-01-18T05:05:22.652532",
  "error_message": null
}
```

**Status Values:**
- `"pending"` - Project created, waiting to start
- `"generating"` - Conversion in progress
- `"success"` - Conversion completed successfully
- `"completed_with_errors"` - Completed with warnings
- `"failed"` - Conversion failed

**Status Codes:**
- `200 OK` - Project found
- `404 Not Found` - Project ID doesn't exist

**Example:**
```bash
curl http://localhost:8000/api/projects/7/status
```

**Polling Recommendation:**
- Poll every 5-10 seconds while `status` is `"pending"` or `"generating"`
- Stop polling once status is `"success"`, `"completed_with_errors"`, or `"failed"`

---

### List Projects

Get a paginated list of all projects.

**Endpoint:** `GET /api/projects`

**Query Parameters:**
- `skip`: Integer (optional, default: 0) - Number of records to skip
- `limit`: Integer (optional, default: 100) - Maximum records to return

**Response:** `200 OK`
```json
{
  "projects": [
    {
      "id": 7,
      "name": "samsung-test-enhanced",
      "status": "success",
      "components_generated": 6,
      "components_reused": 0,
      "created_at": "2026-01-18T05:05:22.652532"
    },
    {
      "id": 6,
      "name": "samsung-strict",
      "status": "success",
      "components_generated": 3,
      "components_reused": 0,
      "created_at": "2026-01-17T19:35:07.615852"
    }
  ],
  "total": 7
}
```

**Status Codes:**
- `200 OK` - Success

**Example:**
```bash
# Get first 10 projects
curl "http://localhost:8000/api/projects?skip=0&limit=10"

# Get next 10 projects (pagination)
curl "http://localhost:8000/api/projects?skip=10&limit=10"
```

---

### Delete Project

Delete a project and its data.

**Endpoint:** `DELETE /api/projects/{project_id}`

**Path Parameters:**
- `project_id`: Integer - Project ID to delete

**Response:** `200 OK`
```json
{
  "message": "Project deleted"
}
```

**Status Codes:**
- `200 OK` - Project deleted
- `404 Not Found` - Project doesn't exist

**Example:**
```bash
curl -X DELETE http://localhost:8000/api/projects/7
```

**Note:** This only deletes the database record, not the generated files on disk.

---

### List Components

Get all components in the reusable component library.

**Endpoint:** `GET /api/components`

**Query Parameters:**
- `category`: String (optional) - Filter by component category
- `limit`: Integer (optional, default: 100) - Maximum records to return

**Response:** `200 OK`
```json
{
  "components": [
    {
      "id": "comp_123",
      "name": "Header",
      "category": "navigation",
      "code": "import React from 'react'...",
      "description": "Navigation header with logo and menu",
      "created_at": "2026-01-18T10:37:00",
      "usage_count": 3
    }
  ],
  "total": 15
}
```

**Status Codes:**
- `200 OK` - Success

**Example:**
```bash
# Get all components
curl http://localhost:8000/api/components

# Filter by category
curl "http://localhost:8000/api/components?category=navigation&limit=50"
```

---

### Get Statistics

Get platform-wide statistics.

**Endpoint:** `GET /api/stats`

**Response:** `200 OK`
```json
{
  "total_projects": 7,
  "completed_projects": 6,
  "total_components": 15,
  "total_component_reuses": 23
}
```

**Status Codes:**
- `200 OK` - Success

**Example:**
```bash
curl http://localhost:8000/api/stats
```

---

## Data Models

### Project Model

```typescript
interface Project {
  id: number;
  name: string;
  figma_url: string;
  status: "pending" | "generating" | "success" | "completed_with_errors" | "failed";
  project_path: string | null;
  components_generated: number;
  components_reused: number;
  conversion_time_seconds: number | null;
  created_at: string;  // ISO 8601 format
  updated_at: string;  // ISO 8601 format
  error_message: string | null;
}
```

### Component Model

```typescript
interface Component {
  id: string;
  name: string;
  category: string;
  code: string;
  description: string;
  created_at: string;
  usage_count: number;
  metadata: Record<string, any>;
}
```

---

## Error Handling

All error responses follow this format:

```json
{
  "detail": "Error message here"
}
```

### Common HTTP Status Codes

- `200 OK` - Request successful
- `400 Bad Request` - Invalid request parameters
- `404 Not Found` - Resource not found
- `500 Internal Server Error` - Server error

### Error Response Examples

**400 Bad Request:**
```json
{
  "detail": "Project 'my-website' already exists"
}
```

**404 Not Found:**
```json
{
  "detail": "Project not found"
}
```

---

## Examples

### Complete Frontend Workflow

```javascript
// 1. Create a new project
const createResponse = await fetch('http://localhost:8000/api/projects/create', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    figma_url: 'https://www.figma.com/design/abc123/MyDesign',
    project_name: 'my-website',
    ui_library: 'tailwind'
  })
});

const { project_id } = await createResponse.json();
console.log('Project ID:', project_id);

// 2. Poll for status
const pollStatus = async () => {
  const statusResponse = await fetch(
    `http://localhost:8000/api/projects/${project_id}/status`
  );
  const status = await statusResponse.json();

  console.log('Status:', status.status);

  if (status.status === 'generating' || status.status === 'pending') {
    // Still processing, poll again in 5 seconds
    setTimeout(pollStatus, 5000);
  } else if (status.status === 'success') {
    console.log('✅ Conversion complete!');
    console.log('Components generated:', status.components_generated);
    console.log('Project path:', status.project_path);
  } else if (status.status === 'failed') {
    console.error('❌ Conversion failed:', status.error_message);
  }
};

pollStatus();

// 3. Get statistics
const statsResponse = await fetch('http://localhost:8000/api/stats');
const stats = await statsResponse.json();
console.log('Platform stats:', stats);
```

### React Hook Example

```typescript
import { useState, useEffect } from 'react';

interface ProjectStatus {
  id: number;
  name: string;
  status: string;
  components_generated: number;
  conversion_time_seconds: number | null;
  error_message: string | null;
}

export function useProjectStatus(projectId: number | null) {
  const [status, setStatus] = useState<ProjectStatus | null>(null);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!projectId) return;

    const fetchStatus = async () => {
      setLoading(true);
      const response = await fetch(
        `http://localhost:8000/api/projects/${projectId}/status`
      );
      const data = await response.json();
      setStatus(data);
      setLoading(false);
    };

    fetchStatus();

    // Poll every 5 seconds if still generating
    const interval = setInterval(() => {
      if (status?.status === 'generating' || status?.status === 'pending') {
        fetchStatus();
      }
    }, 5000);

    return () => clearInterval(interval);
  }, [projectId, status?.status]);

  return { status, loading };
}
```

---

## Additional Resources

- **API Documentation (Interactive):** `http://localhost:8000/docs` - Swagger UI
- **OpenAPI Spec:** `http://localhost:8000/openapi.json`
- **GitHub Repository:** [Link to repo]

---

## Support

For issues or questions:
- Check server logs at the terminal running `uvicorn`
- Verify `.env` configuration (especially `FIGMA_TOKEN`)
- Ensure all dependencies are installed: `uv pip install -r requirements.txt`

---

## Changelog

### v1.0.0 (2026-01-18)
- Initial API release
- Enhanced Figma data extraction
- Image download support
- Component reuse system
- Multi-UI library support (Tailwind, MUI, Chakra)
