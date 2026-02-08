# Plan: Co-Design Skill Test

**Generated**: 2026-02-08

## Overview
A minimal test plan to verify the co-design skill correctly routes design tasks to `claude -p` and standard tasks to Task tool subagents. Creates a tiny demo project with one frontend component and one utility function.

## Prerequisites
- None (pure file creation test)

## Dependency Graph

```
T1 ──┐
     ├── T3
T2 ──┘
```

## Tasks

### T1: Create a styled greeting card component
- **depends_on**: []
- **location**: /home/willr/Applications/swarms/test-output/components/GreetingCard.html
- **description**: Create a single HTML file containing a styled greeting card component. The card should have a title "Hello, Co-Design!", a subtitle "This was built by a design agent", and a gradient background (purple to blue). Use inline CSS. The card should be centered on the page, have rounded corners (16px), a subtle box shadow, white text, and padding of 2rem. Include a simple hover effect that slightly scales the card up.
- **validation**: File exists at the specified path and contains valid HTML with inline CSS styling
- **status**: Completed
- **log**: Created styled greeting card HTML with purple-to-blue gradient, centered layout, rounded corners (16px), box shadow, white text, 2rem padding, and hover scale effect.
- **files edited/created**: test-output/components/GreetingCard.html (created)

### T2: Create a utility function module
- **depends_on**: []
- **location**: /home/willr/Applications/swarms/test-output/utils/helpers.js
- **description**: Create a JavaScript utility module that exports three functions: (1) `formatDate(date)` - takes a Date object and returns a string in "YYYY-MM-DD" format, (2) `capitalize(str)` - capitalizes the first letter of a string, (3) `slugify(str)` - converts a string to a URL-friendly slug (lowercase, spaces to hyphens, remove special chars). Use ES module syntax (export).
- **validation**: File exists and contains three exported functions with correct logic
- **status**: Not Completed
- **log**:
- **files edited/created**:

### T3: Create an index page that imports both
- **depends_on**: [T1, T2]
- **location**: /home/willr/Applications/swarms/test-output/index.html
- **description**: Create an index.html page that includes the greeting card component and demonstrates the utility functions. The page should have a clean layout with the greeting card displayed prominently at the top, and below it a section showing the output of each utility function with example inputs. Style the page with a light gray background, centered content (max-width 800px), and clean typography.
- **validation**: File exists, references both the component and utilities, has proper styling
- **status**: Not Completed
- **log**:
- **files edited/created**:

## Parallel Execution Groups

| Wave | Tasks | Can Start When | Routing |
|------|-------|----------------|---------|
| 1 | T1, T2 | Immediately | T1: design (claude -p), T2: standard (subagent) |
| 2 | T3 | Wave 1 complete | T3: design (claude -p) |

## Testing Strategy
- Verify T1 was routed to `claude -p` (check for log file at /tmp/co-design-T1-output.log)
- Verify T2 was routed to Task tool subagent
- Verify T3 was routed to `claude -p`
- Verify all output files exist and contain expected content

## Risks & Mitigations
- `claude -p` may not be available: Check that `claude` CLI is on PATH
- Output directory may not exist: Create /home/willr/Applications/swarms/test-output/ first
