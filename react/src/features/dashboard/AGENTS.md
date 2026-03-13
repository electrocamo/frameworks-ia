# AGENTS.md — Feature: Dashboard

> Extends the root `AGENTS.md`. Read that file first, then this one.
> Rules here **override** the root document for this feature only.

---

## Context

Main dashboard with multiple real-time data widgets.
Highest fetch volume in the app — queries are optimized with `staleTime` and `refetchInterval`.

---

## ⚠️ Additional Skills for This Feature

```
Are you adding a new widget that fetches server data?
→ READ FIRST: ia/skills/state-and-data/SKILL.md  (section: TanStack Query patterns)

Does the widget include filter forms?
→ READ FIRST: ia/skills/forms-and-validation/SKILL.md

Is the widget causing performance or re-render issues?
→ READ FIRST: ia/skills/performance/SKILL.md
```

---

## Dashboard-Specific Rules

**Widgets:**
- Each widget is a self-contained component with its own `useQuery`
- Maximum one level of composition: `DashboardPage` → `Widget` → UI sub-components
- All widgets use `Suspense` + `ErrorBoundary` — never handle loading/error manually inside a widget

**Queries:**
- `staleTime: 5 * 60 * 1000` (5 min) for low-frequency data
- `refetchInterval: 30_000` (30 sec) for near-real-time data
- Prefetch critical queries in the route loader before the page mounts

**Charts:**
- Library: **Recharts** — do not install alternatives without an ADR
- Always include a descriptive `aria-label` on the chart container element

---

## Feature Structure

```
src/features/dashboard/
  components/
    DashboardPage.tsx
    widgets/
      MetricsWidget.tsx
      MetricsWidget.test.tsx
      ChartWidget.tsx
      ChartWidget.test.tsx
  hooks/
    useMetrics.ts
    useChartData.ts
  services/
    dashboard.service.ts
  types/
    dashboard.types.ts
  index.ts
```
