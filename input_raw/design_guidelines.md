# Design Guidelines: Schakel AI Meeting Automation & Control Center

## Design Approach

**Selected Approach**: Design System - Material Design 3  
**Justification**: This is an information-dense productivity tool requiring clarity, scannability, and robust data handling. Material Design 3 provides excellent patterns for dashboards, tables, forms, and status indicators while maintaining professional aesthetics.

**Key Design Principles**:
- **Clarity First**: Every element serves a functional purpose
- **Data Hierarchy**: Clear visual distinction between primary data (meeting titles, status) and secondary metadata
- **Actionability**: CTAs and interactive elements are immediately identifiable
- **Status Transparency**: Pipeline states, errors, and progress are instantly recognizable

---

## Typography

**Font Stack**: Google Fonts via CDN
- **Primary**: Inter (headings, UI elements, data)
- **Monospace**: JetBrains Mono (logs, timestamps, IDs)

**Hierarchy**:
- **Page Titles**: text-2xl font-semibold (Dashboard, Meeting Details)
- **Section Headers**: text-lg font-medium (Table headers, card titles)
- **Body/Data**: text-sm font-normal (table cells, metadata)
- **Labels**: text-xs font-medium uppercase tracking-wide (form labels, status badges)
- **Monospace Data**: text-sm (timestamps, meeting IDs, error codes)

---

## Layout System

**Spacing Primitives**: Tailwind units of **2, 4, 6, 8, 12, 16**
- **Component padding**: p-4, p-6
- **Section spacing**: gap-6, gap-8
- **Card spacing**: p-6
- **Table cells**: px-4 py-3
- **Form elements**: space-y-4

**Grid Structure**:
- **Container**: max-w-7xl mx-auto px-4
- **Dashboard**: Single column with full-width cards/tables
- **Detail Pages**: Two-column layout (8/4 split) - main content left, metadata/actions right
- **Forms**: Single column max-w-2xl

---

## Component Library

### Navigation
- **Top Navigation Bar**: Sticky header with logo, app title, user profile
- **Sidebar** (optional for multi-section apps): Fixed left sidebar with icon-based navigation
- **Breadcrumbs**: Text-based hierarchy (Dashboard > Meeting Details > [Title])

### Core UI Elements
- **Buttons**: Rounded corners (rounded-md), distinct sizes (sm, md, lg)
  - Primary actions: Filled solid style
  - Secondary: Outlined style
  - Tertiary: Ghost/text style
- **Inputs**: Standard height (h-10), rounded borders (rounded-md), clear focus states
- **Badges**: Pill-shaped (rounded-full px-3 py-1 text-xs) for status indicators (Success, Error, Pending, Manual)
- **Cards**: Elevated surface (shadow-sm border), rounded-lg, consistent p-6

### Data Display
- **Tables**: 
  - Striped rows for scannability
  - Sticky header on scroll
  - Sortable columns with icon indicators
  - Row hover states
  - Action buttons in final column (icon-only with tooltips)
  - Responsive: Stack to cards on mobile
  
- **Status Indicators**: 
  - Dot + text combination (leading colored circle)
  - Badge style for compact areas
  - Progress bars for pipeline stages
  
- **Metadata Grids**: 
  - Two-column layout (label: value)
  - Dividers between sections
  - Monospace for technical values

### Forms
- **Input Fields**: Full-width within container, clear labels above, helper text below, error states with messages
- **Textareas**: Auto-expanding for transcript input
- **Dropdowns**: Native-styled with clear selected state
- **File Uploads**: Drag-and-drop zone with manual browse fallback

### Overlays
- **Modals**: Centered, max-w-lg, backdrop blur
- **Toasts**: Top-right positioned, auto-dismiss, icon + message
- **Tooltips**: Minimal, on-hover for icon buttons and truncated text

### Specialized Components
- **Pipeline Timeline**: Vertical stepper showing stages (Ingest → LLM → SharePoint → Outlook → Productive → Teams)
  - Icons for each stage
  - Status indicators (completed/error/pending)
  - Timestamps and duration
  - Expandable error details

- **Meeting Detail Layout**:
  - **Hero Section**: Meeting title, date, participants (compact, not full viewport)
  - **Tabs**: For switching between Notes, Transcript, Tasks, Logs
  - **Action Bar**: Sticky at top with retry buttons
  
- **Transcript Display**: 
  - Collapsible/expandable sections
  - Monospace font
  - Max-height with scroll

---

## Animations

**Philosophy**: Minimal and purposeful only

**Permitted**:
- Button hover/active states (subtle scale/opacity)
- Loading spinners for async operations
- Toast slide-in/fade-out
- Smooth collapse/expand (height transitions)

**Forbidden**:
- Page transition effects
- Decorative animations
- Scroll-triggered effects
- Complex micro-interactions

---

## Accessibility

- **Keyboard Navigation**: Full support for tab navigation, enter/space activation
- **ARIA Labels**: All icon buttons and interactive elements
- **Focus States**: Clear visible outline on all interactive elements
- **Contrast**: Maintain WCAG AA standards for text and interactive elements
- **Screen Reader**: Proper heading hierarchy, semantic HTML

---

## Images

**No decorative imagery** - this is a utility application. The only visual assets are:
- **Schakel AI Logo**: Top-left navigation (SVG, modest size ~32px height)
- **Icons**: Heroicons library via CDN for UI elements (outline style for secondary actions, solid for primary)
- **Empty States**: Simple icon + text for empty tables/no data scenarios