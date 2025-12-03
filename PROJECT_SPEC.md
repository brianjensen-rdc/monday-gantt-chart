# Monday.com Gantt Chart - Project Specification

## 📋 Project Overview

A Gantt chart-style workback schedule that tracks client projects, resources, timelines, project segments, and deadlines by syncing data from Monday.com dashboards through a Monday.com MCP server.

**Data Source:** Monday.com board "Duplicate of ✅ Active Projects" (35 projects currently, scalable to 75+)

**Primary Goal:** Visualize project timelines, resources, and deliverables in an interactive Gantt chart with filtering and hierarchical views.

---

## 🏗️ Architecture Overview

### Phase 1: Read-Only Gantt Chart (Current)
- Fetch data from Monday.com via MCP server (read-only)
- Display projects and subitems in hierarchical Gantt chart
- Implement filtering, grouping, and timeline visualization
- No database required (pure frontend)

### Phase 2: Bi-Directional Sync (Future)
- Add Supabase for backend functionality
- Create Edge Functions to write data back to Monday.com API
- Enable drag-to-update timelines, status changes, resource reassignment
- Store API token securely in Supabase
- Optional: caching, user preferences, audit logs

---

## 📊 Data Structure

### Monday.com Board Structure

**Board ID:** To be confirmed via MCP  
**Board Name:** "Duplicate of ✅ Active Projects"  
**Item Count:** 35 parent items (projects), multiple subitems per project

### Parent Level (Projects)

| Column Name | Type | Values | Usage |
|------------|------|--------|-------|
| Project Name | text | e.g., "eBook - 'Modernize your .NET and Java Apps on Azure'" | Primary identifier |
| Account Mgr | people | Person object | Display avatar/name |
| Project Mgr | people | Person object | Display avatar/name |
| Creative Lead | people | people | Display avatar/name |
| Creative Brief | file | File attachment | Link icon (optional) |
| Asset Type | dropdown/status | eBook, Video, Social, Newsletter, Blog, Glossary Page, Customer Story, Flyer, Deck Course, Graphic Design, N/A | **Primary color-coding** |
| Status | status | Executing, PO Open, Closing Activities, Done, etc. | Status badge |
| Delivery Date | date | Single date | Parent milestone |
| Total Timeline | timeline | Date range (start - end) | **Calculated from subitems if null** |
| Total Duration | number | Days | Display in info panel |
| Total Estimated Hours | number | Hours | Display in info panel |
| Actual Hours | number | Hours | Display in info panel |

### Subitem Level (Tasks/Deliverables)

| Column Name | Type | Values | Usage |
|------------|------|--------|-------|
| Task Name | text | e.g., "Project Kick-off Meeting: MSFT Review..." | Display in hierarchy |
| Assigned To | people | Can be multiple people | Display avatars |
| Status | status | Done, Client Review, Inactive, etc. | Status badge |
| Timeline | timeline | Date range (start - end) | **Timeline bar visualization** |
| Duration | number | Days | Display on bar |
| Estimated Hours | number | Hours | Optional display |
| Daily Effort | number | Hours/day | Internal calculation |
| Actual Hours | number | Hours | Optional display |
| Dependency | dependency | Links to other tasks | Phase 2 feature |

---

## 🎨 Visual Design

### Color Scheme (Asset Type)
Match Monday.com's color palette:

- **eBook:** `#00C2C2` (Teal)
- **Video:** `#FF6B35` (Orange)
- **Social:** `#E879F9` (Pink/Magenta)
- **Newsletter:** `#66C98E` (Green)
- **Blog:** `#7B68EE` (Purple)
- **Glossary Page:** `#FFB84D` (Yellow)
- **Customer Story:** `#4ECDC4` (Cyan)
- **Flyer:** `#FF6B9D` (Rose)
- **Deck Course:** `#9B59B6` (Violet)
- **Graphic Design:** `#3498DB` (Blue)
- **N/A:** `#95A5A6` (Gray)

### Status Colors
- **Executing:** `#FDAB3D` (Yellow/Orange)
- **PO Open:** `#66C98E` (Green)
- **Closing Activities:** `#9B59B6` (Purple)
- **Done:** `#00C875` (Bright Green)
- **Client Review:** `#E879F9` (Pink)
- **Inactive:** `#C4C4C4` (Gray)

### Timeline Bars
- Default: `#E879F9` (Pink/Magenta) - matching Monday.com
- Hover: Slightly darker shade
- Display date range on bar: "Oct 31 - Dec 19"

---

## 🛠️ Component Architecture

```
/App.tsx
  └─ GanttChartContainer
      ├─ FilterPanel
      │   ├─ AssetTypeFilter (multi-select)
      │   ├─ StatusFilter (multi-select)
      │   ├─ ClientFilter (multi-select)
      │   ├─ ResourceFilter (people picker)
      │   └─ DateRangeFilter (optional)
      │
      ├─ ControlBar
      │   ├─ RefreshButton
      │   ├─ ClearFiltersButton
      │   ├─ ExpandAllButton
      │   └─ CollapseAllButton
      │
      └─ GanttChart
          ├─ TimelineHeader (month/week grid)
          ├─ ProjectList
          │   └─ ProjectRow (repeating)
          │       ├─ ProjectInfo (name, type badge, people)
          │       ├─ TimelineBar (parent timeline)
          │       └─ SubitemList (collapsible)
          │           └─ SubitemRow (repeating)
          │               ├─ SubitemInfo (name, status, people)
          │               └─ TimelineBar (task timeline)
          │
          └─ ResourceView (separate tab/view)
              └─ ResourceRow (grouped by person)
                  └─ TaskBars (all tasks for that person)
```

---

## 🔌 MCP Integration

### Monday.com MCP Server
**Server ID:** `e5be7d52-511c-4782-877f-f20fecf3a6bf`  
**Capabilities:** Read-only access to Monday.com data

### Available Tools

1. **`get_board_items_page`** - Fetch paginated items from board
   - **Input:** `board_id`, `limit`, `cursor`
   - **Output:** Items array with column values, subitems
   - **Usage:** Primary data fetching method

2. **`get_board_info`** - Get board metadata
   - **Input:** `board_id`
   - **Output:** Board name, columns, groups
   - **Usage:** Validate board structure, get column IDs

3. **`search_items`** - Search across boards (optional)
   - **Input:** Query string
   - **Output:** Matching items
   - **Usage:** Future feature for global search

4. **`get_graphql_schema`** - View GraphQL schema
   - **Input:** `operationType` (read/write)
   - **Output:** Available queries/mutations
   - **Usage:** Documentation reference

### Known Limitations

1. **Read-Only:** No write operations exposed via MCP tools
2. **Pagination Required:** For datasets >50 items
3. **Unsupported Columns:** Some aggregated fields not available
4. **Rate Limiting:** Unknown limits, assume conservative usage
5. **No Real-Time Updates:** Manual refresh required

### Data Fetching Strategy

```javascript
// Pseudo-code
async function fetchAllProjects() {
  let allItems = [];
  let cursor = null;
  
  do {
    const response = await run_mcp_tool({
      tool_name: "mcp__monday_tool__get_board_items_page",
      board_id: ACTIVE_PROJECTS_BOARD_ID,
      limit: 50,
      cursor: cursor
    });
    
    allItems = [...allItems, ...response.items];
    cursor = response.cursor;
  } while (cursor);
  
  return processProjectData(allItems);
}
```

---

## ✨ Phase 1 Features (Read-Only)

### Core Functionality
- [x] Connect to Monday.com via MCP
- [x] Fetch all Active Projects with subitems
- [x] Display hierarchical project tree
- [x] Render timeline bars for projects and tasks
- [x] Calculate parent timelines from subitems when null
- [x] Color-code by Asset Type
- [x] Show status badges
- [x] Display people avatars
- [x] Expand/collapse project hierarchy

### Filtering
- [x] Multi-select Asset Type filter
- [x] Multi-select Status filter
- [x] Multi-select Client filter
- [x] Multi-select Resource filter (people)
- [x] Date range filter (optional)
- [x] Clear all filters button

### Timeline Features
- [x] Auto-scale timeline to fit date range
- [x] Month/week grid headers
- [x] Today indicator line
- [x] Hover tooltips showing full details
- [x] Date labels on timeline bars
- [x] Duration display

### UI/UX
- [x] Manual refresh button
- [x] Loading states
- [x] Empty states (no projects, no filters matched)
- [x] Responsive layout (desktop-first, mobile-friendly)
- [x] Smooth expand/collapse animations
- [x] Professional styling matching Monday.com aesthetic

### Resource View (Separate Tab)
- [x] Group tasks by assigned person
- [x] Show all tasks for each resource
- [x] Identify overallocation conflicts
- [x] Timeline bars per resource

---

## 🚀 Phase 2 Features (Bi-Directional Sync)

**Requires:** Supabase setup

### Write Operations (via Supabase Edge Functions)
- [ ] Drag timeline bars to update dates
- [ ] Click to change status
- [ ] Reassign resources via dropdown
- [ ] Add new tasks (subitems)
- [ ] Add new projects (parent items)
- [ ] Update task descriptions/notes
- [ ] Mark tasks complete

### Backend (Supabase)
- [ ] Edge Function: `updateTaskTimeline`
- [ ] Edge Function: `updateTaskStatus`
- [ ] Edge Function: `updateTaskResource`
- [ ] Edge Function: `createTask`
- [ ] Edge Function: `createProject`
- [ ] Secure API token storage (Monday.com)
- [ ] User authentication (optional)
- [ ] Data caching layer (optional)
- [ ] Audit log table (optional)

### Advanced Features
- [ ] Optimistic UI updates
- [ ] Conflict resolution (if data changes externally)
- [ ] Undo/redo functionality
- [ ] Bulk edit operations
- [ ] Export to PDF/Excel
- [ ] Email notifications on changes

---

## 🔒 Security Considerations

### Phase 1 (No Supabase)
- ✅ No API keys in frontend code
- ✅ MCP handles authentication
- ✅ Read-only access (no data modification risk)

### Phase 2 (With Supabase)
- [ ] Store Monday.com API token in Supabase secrets (not in code)
- [ ] Use Row Level Security (RLS) in Supabase
- [ ] Validate all inputs on Edge Functions
- [ ] Rate limit API calls to Monday.com
- [ ] Log all write operations for audit trail
- [ ] **Note:** Figma Make not suitable for PII or highly sensitive data

---

## 📝 Technical Decisions

### Technology Stack
- **Framework:** React (via Figma Make)
- **Styling:** Tailwind CSS v4.0
- **Icons:** lucide-react
- **Data Fetching:** MCP tools (Monday.com server)
- **State Management:** React useState/useContext
- **Date Library:** date-fns or native Date objects
- **Backend (Phase 2):** Supabase (Edge Functions, Database, Auth)

### Libraries to Use
- `lucide-react` - Icons
- `date-fns` - Date manipulation
- `recharts` - Timeline visualization (if needed)
- Motion (`motion/react`) - Animations

### Libraries to Avoid
- ❌ `react-gantt-timeline` - Not needed, custom implementation preferred
- ❌ `konva` - Not supported in Figma Make
- ❌ `react-resizable` - Use `re-resizable` instead if needed

---

## 🎯 Success Metrics

### Phase 1
- Displays all 35 active projects correctly
- Loads in <3 seconds
- Filters apply in <500ms
- No visual regressions from Monday.com data
- Responsive on desktop (1920px+) and laptop (1366px+)

### Phase 2
- Bi-directional sync completes in <2 seconds
- 99% data accuracy (matches Monday.com)
- No conflicts or data loss
- User-friendly error messages

---

## 🐛 Known Issues & Workarounds

### Issue 1: Parent Timeline Null
**Problem:** Some projects have no timeline set at parent level  
**Workaround:** Calculate from min(subitem start dates) to max(subitem end dates)

### Issue 2: Unscheduled Tasks
**Problem:** Some tasks have no timeline  
**Workaround:** Display in separate "Unscheduled" section or hide with filter option

### Issue 3: MCP Pagination
**Problem:** Large datasets require multiple API calls  
**Workaround:** Implement cursor-based pagination with loading indicator

### Issue 4: Column Type Variations
**Problem:** Monday.com column types may vary (text vs dropdown for client)  
**Workaround:** Handle both types gracefully with type checking

---

## 📅 Implementation Timeline

### Immediate (Today)
1. ✅ Create project specification
2. ✅ Set up GitHub repository
3. ⏳ Build core Gantt chart components
4. ⏳ Implement MCP data fetching
5. ⏳ Add filtering system
6. ⏳ Test with live Monday.com data

### Next Session
- Polish UI/UX based on feedback
- Add resource allocation view
- Optimize performance
- Handle edge cases

### Future (Phase 2)
- Set up Supabase project
- Build Edge Functions for write operations
- Implement drag-to-update functionality
- Add comprehensive error handling

---

## 🔗 Resources

- **Monday.com API Docs:** https://developer.monday.com/api-reference/docs
- **Monday.com GraphQL Explorer:** https://monday.com/developers/v2/try-it-yourself
- **Supabase Docs:** https://supabase.com/docs
- **Figma Make Docs:** (internal)

---

## 📞 Questions & Decisions Log

### Confirmed Decisions
1. ✅ Use phased approach (read-only first, then bi-directional)
2. ✅ Color-code by Asset Type (not client)
3. ✅ Calculate parent timelines from subitems when missing
4. ✅ Use Supabase for Phase 2 bi-directional sync
5. ✅ GitHub integration for version control
6. ✅ Desktop-first responsive design

### Pending Decisions
- [ ] Confirm exact Monday.com board ID
- [ ] Decide on default date range for timeline view (3 months? 6 months? All time?)
- [ ] Resource view: separate page or toggle view?
- [ ] Handle dependencies visualization (Phase 1 or Phase 2?)

---

**Document Version:** 1.0  
**Last Updated:** December 3, 2024  
**Author:** AI Assistant (Figma Make)  
**Project Owner:** brianjensen-rdc
