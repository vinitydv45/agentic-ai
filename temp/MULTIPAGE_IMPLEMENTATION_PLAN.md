# Multi-Page Support Implementation Plan

## Overview

Enable users to choose whether a new Figma design should be:
- **Option A**: Added as a NEW PAGE in an existing React project (React Router, shared codebase)
- **Option B**: Created as a NEW PROJECT directory (current behavior)

**Critical Requirement**: Component reuse MUST work in BOTH scenarios.

---

## API Changes

### 1. Add New Parameter: `add_as`

Add to both endpoints:
- `POST /api/projects/create`
- `POST /api/projects/add-website`

**Request Body Schema**:
```json
{
  "figma_url": "string (required)",
  "project_name": "string (required)",
  "ui_library": "string (optional, default: tailwind)",
  "add_as": "string (optional, default: new_project)",
  "parent_project_id": "integer (required if add_as=new_page)"
}
```

**Valid Values for `add_as`**:
- `"new_project"` (default) - Create separate React app directory
- `"new_page"` - Add as page in existing project (requires `parent_project_id`)

### 2. Validation Rules

```python
if request.add_as == "new_page":
    # Validate parent_project_id exists
    if not request.parent_project_id:
        raise HTTPException(400, "parent_project_id required when add_as=new_page")

    parent_project = store.get(request.parent_project_id)
    if not parent_project:
        raise HTTPException(404, f"Parent project {request.parent_project_id} not found")

    if parent_project.status != "success":
        raise HTTPException(400, "Parent project must be completed before adding pages")

    # Validate project_name is unique as page name within parent
    # (will be used as route path: /project_name)
```

### 3. Database Schema Addition

Add to `Project` model in `backend/database/models.py`:

```python
class Project(Base):
    # ... existing fields ...

    # New fields for multi-page support
    parent_project_id: Mapped[Optional[int]] = mapped_column(
        ForeignKey("projects.id"),
        nullable=True
    )
    is_page: Mapped[bool] = mapped_column(default=False)
    route_path: Mapped[Optional[str]] = mapped_column(nullable=True)

    # Relationships
    parent: Mapped[Optional["Project"]] = relationship(
        "Project",
        remote_side="Project.id",
        back_populates="pages"
    )
    pages: Mapped[List["Project"]] = relationship(
        "Project",
        back_populates="parent"
    )
```

---

## Architecture Changes

### 1. Project Structure Comparison

**Current (Option B - new_project)**:
```
generated_projects/
├── project-1/
│   ├── public/
│   ├── src/
│   │   └── components/
│   │       ├── Header.tsx
│   │       └── Footer.tsx
│   └── package.json
└── project-2/
    ├── public/
    └── src/
```

**New (Option A - new_page)**:
```
generated_projects/
└── project-1/
    ├── public/
    │   └── images/
    │       ├── page1-hero.png
    │       └── page2-banner.png
    ├── src/
    │   ├── components/
    │   │   ├── Header.tsx      ← Shared across pages
    │   │   └── Footer.tsx      ← Shared across pages
    │   ├── pages/
    │   │   ├── HomePage.tsx    ← First Figma
    │   │   └── AboutPage.tsx   ← Second Figma (new page)
    │   └── App.tsx             ← React Router setup
    └── package.json
```

### 2. React Router Template Setup

**Update `templates/react-tailwind/src/App.tsx`**:

```typescript
import React from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';

// Pages will be generated here
import HomePage from './pages/HomePage';

function App() {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<HomePage />} />
        {/* Additional routes will be added here */}
      </Routes>
    </Router>
  );
}

export default App;
```

**Add React Router dependency to `package.json`**:
```json
{
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.28.0"
  }
}
```

---

## Agent Behavior Changes

### 1. Update `convert_figma_to_react()` Method

**File**: `backend/agents/figma_to_react.py`

```python
async def convert_figma_to_react(
    self,
    figma_url: str,
    project_name: str,
    is_new_project: bool = True,
    ui_library: str = "tailwind",
    add_as: str = "new_project",         # NEW
    parent_project_path: Optional[Path] = None,  # NEW
) -> dict:
    """
    Convert Figma design to React code.

    Args:
        add_as: "new_project" or "new_page"
        parent_project_path: Required if add_as="new_page"
    """
```

### 2. Conditional Project Setup

```python
if add_as == "new_page":
    # Use existing project directory
    project_dir = parent_project_path
    pages_dir = project_dir / "src" / "pages"
    pages_dir.mkdir(exist_ok=True)

    # Convert project_name to PascalCase for component name
    page_component_name = "".join(word.capitalize() for word in project_name.split("-"))
    route_path = f"/{project_name}"

else:  # new_project
    # Current behavior: create new directory
    project_dir = settings.generated_projects_dir / project_name
    if project_dir.exists():
        shutil.rmtree(project_dir)

    # Copy React template
    shutil.copytree(
        settings.templates_dir / f"react-{ui_library}",
        project_dir,
    )
```

### 3. Update System Prompts

**For `add_as="new_page"` mode**:

```python
system_prompt = f"""You are an expert React + TypeScript + {ui_library.upper()} developer.

CRITICAL: You are adding a NEW PAGE to an EXISTING React project.

PROJECT CONTEXT:
- Parent project: {parent_project_path.name}
- New page name: {page_component_name}
- Route path: {route_path}
- Existing components: {existing_components}

TASK:
1. Create a NEW PAGE COMPONENT in src/pages/{page_component_name}.tsx
2. REUSE existing components from src/components/ whenever possible (>80% similarity)
3. Only create NEW components if no similar component exists
4. Import and use shared components
5. Update src/App.tsx to add the new route

CONSTRAINTS:
- DO NOT modify existing pages
- DO NOT duplicate components that already exist
- DO import and reuse: Header, Footer, Button, etc.
- DO create page-specific sections in the page component
- Use exact design values from Figma data provided

EXISTING COMPONENTS IN PROJECT:
{format_existing_components_list()}

COMPONENT REUSE LIBRARY (cross-project):
{format_library_components()}

OUTPUT STRUCTURE:
1. src/pages/{page_component_name}.tsx - Main page component
2. src/components/NewComponent.tsx - Only if no match in existing/library
3. Updated src/App.tsx - Add route: <Route path="{route_path}" element={{<{page_component_name} />}} />
"""
```

**For `add_as="new_project"` mode**:
```python
# Keep existing system prompt (current behavior)
```

### 4. Component Reuse Strategy

**Two-tier reuse system**:

1. **Within-project reuse (new_page mode)**:
   - Check `src/components/` directory for existing components
   - Direct TypeScript imports: `import { Header } from '../components/Header'`
   - Similarity check using file content embeddings

2. **Cross-project reuse (both modes)**:
   - Check component library database (RAG store)
   - Copy component file if >80% similar
   - Add to project's `src/components/`

```python
async def find_reusable_components(
    description: str,
    project_dir: Optional[Path] = None,
) -> dict:
    """Find reusable components from both project and library."""

    matches = {
        "within_project": [],  # From src/components/ (new_page mode)
        "from_library": [],    # From component library (both modes)
    }

    # 1. Check project's existing components (if new_page mode)
    if project_dir:
        components_dir = project_dir / "src" / "components"
        if components_dir.exists():
            for comp_file in components_dir.glob("*.tsx"):
                similarity = calculate_similarity(description, comp_file.read_text())
                if similarity > 0.8:
                    matches["within_project"].append({
                        "name": comp_file.stem,
                        "path": str(comp_file),
                        "similarity": similarity,
                    })

    # 2. Check component library (RAG store)
    library_matches = component_store.search(
        query=description,
        min_similarity=0.8,
        limit=5,
    )
    matches["from_library"] = library_matches

    return matches
```

---

## File Changes Required

### 1. `backend/database/models.py`

**Add new columns**:
```python
parent_project_id: Mapped[Optional[int]]
is_page: Mapped[bool] = mapped_column(default=False)
route_path: Mapped[Optional[str]]
```

**Migration command**:
```bash
# If using Alembic
alembic revision --autogenerate -m "Add multi-page support fields"
alembic upgrade head
```

### 2. `backend/main.py`

**Update request models**:
```python
class ProjectCreateRequest(BaseModel):
    figma_url: str
    project_name: str
    ui_library: str = "tailwind"
    add_as: str = "new_project"  # NEW
    parent_project_id: Optional[int] = None  # NEW
```

**Update endpoint logic**:
```python
@app.post("/api/projects/create", response_model=ProjectResponse)
async def create_project(request: ProjectCreateRequest, background_tasks: BackgroundTasks):
    # Validate add_as parameter
    if request.add_as not in ["new_project", "new_page"]:
        raise HTTPException(400, "add_as must be 'new_project' or 'new_page'")

    # If new_page, validate parent exists
    parent_project = None
    parent_project_path = None
    if request.add_as == "new_page":
        if not request.parent_project_id:
            raise HTTPException(400, "parent_project_id required for new_page mode")

        parent_project = store.get(request.parent_project_id)
        if not parent_project:
            raise HTTPException(404, "Parent project not found")

        if parent_project.status != "success":
            raise HTTPException(400, "Parent project must be completed")

        parent_project_path = Path(parent_project.project_path)

    # Create project record
    project = store.create(
        name=request.project_name,
        figma_url=request.figma_url,
        parent_project_id=request.parent_project_id,
        is_page=(request.add_as == "new_page"),
        route_path=f"/{request.project_name}" if request.add_as == "new_page" else None,
    )

    # Start background conversion
    background_tasks.add_task(
        run_conversion_sync,
        project_id=project.id,
        figma_url=request.figma_url,
        project_name=request.project_name,
        is_new_project=(store.count() == 1),
        ui_library=request.ui_library,
        add_as=request.add_as,
        parent_project_path=parent_project_path,
    )
```

### 3. `backend/agents/figma_to_react.py`

**Major changes**:
1. Add `add_as` and `parent_project_path` parameters
2. Add `find_reusable_components()` method
3. Add `format_existing_components_list()` method
4. Update system prompt conditionally
5. Update file creation logic (pages/ vs root)
6. Add React Router update logic

### 4. `templates/react-tailwind/package.json`

**Add React Router**:
```json
{
  "dependencies": {
    "react-router-dom": "^6.28.0"
  }
}
```

### 5. `templates/react-tailwind/src/App.tsx`

**Add Router setup**:
```typescript
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import HomePage from './pages/HomePage';

function App() {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<HomePage />} />
      </Routes>
    </Router>
  );
}
```

### 6. Create `templates/react-tailwind/src/pages/` directory

```bash
mkdir -p templates/react-tailwind/src/pages
```

---

## Testing Plan

### Test Case 1: Create New Project (Existing Behavior)
```bash
curl -X POST http://localhost:8000/api/projects/create \
  -H "Content-Type: application/json" \
  -d '{
    "figma_url": "https://www.figma.com/design/a577puJyvBsiQljgPCh4IA/Samsung-Website-Redesign",
    "project_name": "samsung-main",
    "ui_library": "tailwind",
    "add_as": "new_project"
  }'
```

**Expected**:
- Creates `generated_projects/samsung-main/`
- Components: Header, Footer, HeroSection, etc.
- Images downloaded to `public/images/`
- Single page: `src/App.tsx` with all sections
- ✅ Component reuse from library

### Test Case 2: Add New Page to Existing Project
```bash
# Wait for Test Case 1 to complete, get project_id=11

curl -X POST http://localhost:8000/api/projects/create \
  -H "Content-Type: application/json" \
  -d '{
    "figma_url": "https://www.figma.com/design/CT3jTGs5RJ5yQksZvhApof/Samsung-Website-Redesign",
    "project_name": "about-page",
    "ui_library": "tailwind",
    "add_as": "new_page",
    "parent_project_id": 11
  }'
```

**Expected**:
- Uses existing `generated_projects/samsung-main/` directory
- Creates `src/pages/AboutPage.tsx`
- Updates `src/App.tsx` with new route: `<Route path="/about-page" element={<AboutPage />} />`
- ✅ Reuses Header/Footer from `src/components/` (within-project)
- ✅ Reuses other components from library if similar
- Downloads new images to same `public/images/` folder
- Builds successfully with React Router

### Verification Checklist

**For Test Case 1**:
- [ ] Project directory created
- [ ] Components generated
- [ ] Images downloaded
- [ ] `npm run build` succeeds
- [ ] Component saved to library
- [ ] Status = "success"

**For Test Case 2**:
- [ ] NO new project directory created
- [ ] `src/pages/AboutPage.tsx` created
- [ ] `src/App.tsx` updated with route
- [ ] Header/Footer reused from existing components
- [ ] Images added to existing public/images/
- [ ] `npm run build` succeeds with routing
- [ ] Status = "success"
- [ ] Can navigate to http://localhost:5173/ and http://localhost:5173/about-page

---

## Implementation Order

1. **Database Migration** (10 min)
   - Add new columns to `Project` model
   - Run migration

2. **Update API Models** (10 min)
   - Add `add_as` and `parent_project_id` to request models
   - Add validation logic

3. **Update React Template** (15 min)
   - Add React Router to package.json
   - Update App.tsx with Router setup
   - Create src/pages/ directory

4. **Update Agent Logic** (60 min)
   - Add parameters to `convert_figma_to_react()`
   - Implement conditional project setup
   - Add within-project component detection
   - Update system prompts
   - Add React Router file update logic

5. **Update API Endpoints** (20 min)
   - Add validation for new parameters
   - Pass new parameters to background task
   - Update background task signature

6. **Testing** (30 min)
   - Test Case 1: New project
   - Test Case 2: Add page
   - Verify builds
   - Verify routing works

**Total Estimated Time**: ~2.5 hours

---

## Risk Mitigation

### Risk 1: React Router Conflicts
**Mitigation**: Use consistent Router setup, wrap all routes in `<Routes>`

### Risk 2: Component Import Path Issues
**Mitigation**: Use relative imports, maintain consistent directory structure

### Risk 3: Image Path Collisions
**Mitigation**: Prefix image filenames with page name: `about-page-hero.png`

### Risk 4: Build Failures with Routing
**Mitigation**: Test builds after each page addition, validate TypeScript types

---

## Success Criteria

1. ✅ User can create standalone projects (existing behavior preserved)
2. ✅ User can add pages to existing projects
3. ✅ Component reuse works in both modes:
   - Within-project reuse (same codebase)
   - Cross-project reuse (from library)
4. ✅ Images download correctly in both modes
5. ✅ React Router navigation works
6. ✅ Builds succeed with TypeScript compilation
7. ✅ No duplicate components created unnecessarily
8. ✅ API validation prevents invalid requests

---

## Future Enhancements

1. **Page Management UI**: List pages within a project, delete pages
2. **Shared Layout Components**: Global navbar/footer that wraps all pages
3. **Route Grouping**: Nested routes, protected routes
4. **Component Refactoring**: Detect when to extract shared component from page-specific code
5. **Version Control**: Track changes across page additions
