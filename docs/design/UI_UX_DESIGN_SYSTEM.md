# UI/UX Design System
## CompeteDexter Platform

**Version**: 1.0
**Last Updated**: 2026-01-12
**Owner**: Design Team

---

## Design Principles

### 1. **Clarity Over Cleverness**
- Information is dense - make it scannable
- Use plain language, avoid jargon
- Progressive disclosure: show basics, reveal depth on demand

### 2. **Speed of Insight**
- Users need answers fast - minimize clicks
- Streaming responses keep users engaged
- Real-time updates feel immediate

### 3. **Trust Through Transparency**
- Always show sources
- Explain confidence levels
- Make agent's thought process visible

### 4. **Powerful Yet Approachable**
- Advanced features don't overwhelm beginners
- Templates and suggestions guide new users
- Power users can bypass and go deep

### 5. **Data-Dense Without Overwhelming**
- Use visual hierarchy effectively
- White space is strategic
- Color encodes meaning, not decoration

---

## Visual Design

### Color Palette

```css
/* Primary Colors */
--primary-900: #0A1628;    /* Deep blue - headers, emphasis */
--primary-700: #1E3A5F;    /* Brand blue - primary actions */
--primary-500: #2E5C8A;    /* Medium blue - hover states */
--primary-300: #5A8FBF;    /* Light blue - backgrounds */
--primary-100: #E6F0F8;    /* Very light blue - subtle backgrounds */

/* Secondary Colors */
--secondary-900: #4A1D7E;  /* Purple - premium features */
--secondary-500: #7B3FBF;  /* Medium purple - accents */
--secondary-100: #F0E6F8;  /* Light purple - highlights */

/* Neutral Colors */
--neutral-900: #111827;    /* Almost black - body text */
--neutral-700: #374151;    /* Dark gray - secondary text */
--neutral-500: #6B7280;    /* Medium gray - disabled, placeholders */
--neutral-300: #D1D5DB;    /* Light gray - borders */
--neutral-100: #F3F4F6;    /* Very light gray - backgrounds */
--neutral-50: #F9FAFB;     /* Off-white - page background */
--white: #FFFFFF;

/* Semantic Colors */
--success-700: #047857;    /* Green - positive changes */
--success-100: #D1FAE5;    /* Light green - success backgrounds */

--warning-700: #B45309;    /* Orange - warnings */
--warning-100: #FEF3C7;    /* Light orange - warning backgrounds */

--danger-700: #B91C1C;     /* Red - negative changes */
--danger-100: #FEE2E2;     /* Light red - error backgrounds */

--info-700: #1D4ED8;       /* Blue - informational */
--info-100: #DBEAFE;       /* Light blue - info backgrounds */
```

### Typography

```css
/* Font Family */
--font-sans: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
--font-mono: 'SF Mono', 'Monaco', 'Inconsolata', 'Fira Mono', 'Droid Sans Mono', monospace;

/* Font Sizes */
--text-xs: 0.75rem;      /* 12px - labels, captions */
--text-sm: 0.875rem;     /* 14px - body small, metadata */
--text-base: 1rem;       /* 16px - body text */
--text-lg: 1.125rem;     /* 18px - large body */
--text-xl: 1.25rem;      /* 20px - small headings */
--text-2xl: 1.5rem;      /* 24px - section headings */
--text-3xl: 1.875rem;    /* 30px - page headings */
--text-4xl: 2.25rem;     /* 36px - hero text */

/* Font Weights */
--font-normal: 400;
--font-medium: 500;
--font-semibold: 600;
--font-bold: 700;

/* Line Heights */
--leading-tight: 1.25;
--leading-normal: 1.5;
--leading-relaxed: 1.75;
```

### Spacing System

```css
/* Based on 4px grid */
--space-1: 0.25rem;    /* 4px */
--space-2: 0.5rem;     /* 8px */
--space-3: 0.75rem;    /* 12px */
--space-4: 1rem;       /* 16px */
--space-5: 1.25rem;    /* 20px */
--space-6: 1.5rem;     /* 24px */
--space-8: 2rem;       /* 32px */
--space-10: 2.5rem;    /* 40px */
--space-12: 3rem;      /* 48px */
--space-16: 4rem;      /* 64px */
```

### Elevation (Shadows)

```css
--shadow-sm: 0 1px 2px 0 rgba(0, 0, 0, 0.05);
--shadow-md: 0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06);
--shadow-lg: 0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05);
--shadow-xl: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
```

### Border Radius

```css
--radius-sm: 0.25rem;  /* 4px - buttons, inputs */
--radius-md: 0.5rem;   /* 8px - cards */
--radius-lg: 1rem;     /* 16px - modals */
--radius-full: 9999px; /* Fully rounded - pills, avatars */
```

---

## Component Library

### 1. Buttons

**Primary Button**
```jsx
<button className="btn btn-primary">
  Ask Question
</button>
```

```css
.btn {
  padding: var(--space-2) var(--space-4);
  border-radius: var(--radius-sm);
  font-size: var(--text-sm);
  font-weight: var(--font-medium);
  cursor: pointer;
  transition: all 0.15s ease;
}

.btn-primary {
  background: var(--primary-700);
  color: var(--white);
  border: none;
}

.btn-primary:hover {
  background: var(--primary-900);
}

.btn-secondary {
  background: var(--white);
  color: var(--primary-700);
  border: 1px solid var(--neutral-300);
}

.btn-ghost {
  background: transparent;
  color: var(--neutral-700);
}
```

**Sizes:**
- Small: 32px height, 12px padding
- Medium (default): 40px height, 16px padding
- Large: 48px height, 20px padding

---

### 2. Input Fields

**Text Input**
```jsx
<div className="input-group">
  <label className="input-label">Company Name</label>
  <input type="text" className="input" placeholder="Enter competitor name..." />
  <p className="input-help">The official name or domain of the competitor</p>
</div>
```

```css
.input-group {
  margin-bottom: var(--space-4);
}

.input-label {
  display: block;
  font-size: var(--text-sm);
  font-weight: var(--font-medium);
  color: var(--neutral-900);
  margin-bottom: var(--space-2);
}

.input {
  width: 100%;
  padding: var(--space-3) var(--space-4);
  border: 1px solid var(--neutral-300);
  border-radius: var(--radius-sm);
  font-size: var(--text-base);
  color: var(--neutral-900);
  transition: border-color 0.15s ease;
}

.input:focus {
  outline: none;
  border-color: var(--primary-700);
  box-shadow: 0 0 0 3px rgba(46, 92, 138, 0.1);
}

.input-help {
  margin-top: var(--space-2);
  font-size: var(--text-xs);
  color: var(--neutral-500);
}
```

---

### 3. Cards

**Competitor Card**
```jsx
<div className="card">
  <div className="card-header">
    <img src={logo} className="card-logo" />
    <div>
      <h3 className="card-title">Competitor Name</h3>
      <p className="card-subtitle">competitor.com</p>
    </div>
  </div>
  <div className="card-body">
    <p className="card-description">Brief description of competitor...</p>
  </div>
  <div className="card-footer">
    <span className="badge badge-primary">Primary</span>
    <button className="btn-ghost">View Details →</button>
  </div>
</div>
```

```css
.card {
  background: var(--white);
  border: 1px solid var(--neutral-300);
  border-radius: var(--radius-md);
  padding: var(--space-6);
  box-shadow: var(--shadow-sm);
  transition: box-shadow 0.15s ease;
}

.card:hover {
  box-shadow: var(--shadow-md);
}

.card-header {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  margin-bottom: var(--space-4);
}

.card-logo {
  width: 48px;
  height: 48px;
  border-radius: var(--radius-sm);
}

.card-title {
  font-size: var(--text-lg);
  font-weight: var(--font-semibold);
  color: var(--neutral-900);
}

.card-subtitle {
  font-size: var(--text-sm);
  color: var(--neutral-500);
}

.card-footer {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-top: var(--space-4);
  padding-top: var(--space-4);
  border-top: 1px solid var(--neutral-200);
}
```

---

### 4. Badges & Tags

```jsx
<span className="badge badge-primary">Primary</span>
<span className="badge badge-success">Active</span>
<span className="badge badge-warning">Pending</span>
```

```css
.badge {
  display: inline-flex;
  padding: var(--space-1) var(--space-3);
  border-radius: var(--radius-full);
  font-size: var(--text-xs);
  font-weight: var(--font-medium);
}

.badge-primary {
  background: var(--primary-100);
  color: var(--primary-900);
}

.badge-success {
  background: var(--success-100);
  color: var(--success-700);
}

.badge-warning {
  background: var(--warning-100);
  color: var(--warning-700);
}
```

---

### 5. Alert Notifications

```jsx
<div className="alert alert-info">
  <svg className="alert-icon">...</svg>
  <div className="alert-content">
    <h4 className="alert-title">New funding round</h4>
    <p className="alert-description">Competitor XYZ raised $50M Series B</p>
  </div>
  <button className="alert-dismiss">×</button>
</div>
```

```css
.alert {
  display: flex;
  gap: var(--space-3);
  padding: var(--space-4);
  border-radius: var(--radius-md);
  border-left: 4px solid;
}

.alert-info {
  background: var(--info-100);
  border-color: var(--info-700);
}

.alert-success {
  background: var(--success-100);
  border-color: var(--success-700);
}

.alert-warning {
  background: var(--warning-100);
  border-color: var(--warning-700);
}

.alert-title {
  font-size: var(--text-sm);
  font-weight: var(--font-semibold);
  margin-bottom: var(--space-1);
}

.alert-description {
  font-size: var(--text-sm);
  color: var(--neutral-700);
}
```

---

## Page Layouts

### Main Application Layout

```
┌──────────────────────────────────────────────────────────────┐
│  Topbar (60px height)                                         │
│  ┌──────────┐  [Search...]        [Alerts] [User Menu]      │
│  │   Logo   │                                                 │
│  └──────────┘                                                 │
├──────────────┬───────────────────────────────────────────────┤
│              │                                                │
│   Sidebar    │            Main Content Area                  │
│   (240px)    │                                                │
│              │                                                │
│ • Home       │                                                │
│ • Competitors│                                                │
│ • Queries    │                                                │
│ • Alerts     │                                                │
│ • Reports    │                                                │
│              │                                                │
│              │                                                │
│ [Settings]   │                                                │
│ [Help]       │                                                │
│              │                                                │
└──────────────┴───────────────────────────────────────────────┘
```

---

## Key Screens & Wireframes

### 1. Home Dashboard

**Layout:**

```
┌────────────────────────────────────────────────────────────────┐
│  Home Dashboard                                                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Quick Query Box                                          │ │
│  │  ┌────────────────────────────────────────┬────────────┐ │ │
│  │  │ Ask a question about your competitors..│  [Ask →]   │ │ │
│  │  └────────────────────────────────────────┴────────────┘ │ │
│  │                                                           │ │
│  │  Suggested: "What funding did competitors raise?"        │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌─────────────────────┐  ┌─────────────────────────────────┐ │
│  │  Recent Activity    │  │  Quick Stats                    │ │
│  │                     │  │                                 │ │
│  │  🔔 CompanyA raised │  │  📊 12 Competitors tracked     │ │
│  │     $50M Series B   │  │  📝 24 Queries this month      │ │
│  │  📱 CompanyB launched│  │  ⚡ 8 Active alerts           │ │
│  │     mobile app      │  │  📈 15% more intel vs last mo │ │
│  │  💼 CompanyC hired  │  │                                 │ │
│  │     new CTO         │  │                                 │ │
│  └─────────────────────┘  └─────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  Your Competitors                           [+ Add More] │ │
│  │                                                           │ │
│  │  [Card] [Card] [Card] [Card] [Card]                     │ │
│  │  Logo   Logo   Logo   Logo   Logo                       │ │
│  │  Name   Name   Name   Name   Name                       │ │
│  │  Tag    Tag    Tag    Tag    Tag                        │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

**Key Elements:**
1. **Quick Query Box**: Primary CTA - always visible
2. **Recent Activity**: Feed of latest competitive intelligence
3. **Quick Stats**: Overview metrics
4. **Competitor Cards**: Visual grid of tracked competitors

---

### 2. Query Interface (Chat)

**Layout:**

```
┌────────────────────────────────────────────────────────────────┐
│  Competitive Intelligence                       [New Query]    │
├────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  You: What is CompanyA's pricing strategy?               │ │
│  │  10:23 AM                                                 │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  🤖 CompeteDexter                                        │ │
│  │                                                           │ │
│  │  ⏳ Understanding your question...                       │ │
│  │  ✅ Identified: CompanyA, pricing analysis               │ │
│  │  📋 Planning research tasks...                           │ │
│  │  ✅ Created 4 tasks                                      │ │
│  │  🔍 Executing tasks...                                   │ │
│  │     ▶ Fetching pricing page data... ✅                  │ │
│  │     ▶ Analyzing pricing tiers... ⏳                     │ │
│  │     ▶ Comparing to competitors...                       │ │
│  │                                                           │ │
│  │  📊 CompanyA uses a tiered pricing model with 3 tiers:  │ │
│  │                                                           │ │
│  │  1. **Starter**: $49/mo - up to 10 users                │ │
│  │  2. **Professional**: $199/mo - up to 50 users          │ │
│  │  3. **Enterprise**: Custom - unlimited users            │ │
│  │                                                           │ │
│  │  Key insights:                                           │ │
│  │  • Recently increased prices by 15% (3 months ago)      │ │
│  │  • Added usage-based pricing for API calls              │ │
│  │  • Enterprise tier focuses on security & compliance     │ │
│  │                                                           │ │
│  │  vs. Competitors:                                        │ │
│  │  • 20% more expensive than CompanyB                     │ │
│  │  • Similar to CompanyC but with more features           │ │
│  │                                                           │ │
│  │  Sources:                                                │ │
│  │  • companya.com/pricing (accessed today)                │ │
│  │  • Web archive: pricing history                         │ │
│  │  • G2 reviews mentioning pricing                        │ │
│  │                                                           │ │
│  │  10:24 AM                                                │ │
│  │                                                           │ │
│  │  [👍 Helpful] [👎 Not Helpful] [🔗 Share] [💾 Save]   │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  [Type your follow-up question...]            [Send →]   │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

**Key Elements:**
1. **Progress Indicators**: Show agent phases (build trust)
2. **Structured Answer**: Clear hierarchy, scannable
3. **Source Citations**: Transparency and credibility
4. **Feedback Buttons**: Improve over time
5. **Follow-up Input**: Continue conversation

---

### 3. Competitor Profile Page

**Layout:**

```
┌────────────────────────────────────────────────────────────────┐
│  ← Back to Competitors                          [Edit] [⋮]     │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌────────┐  CompanyA                                         │
│  │  Logo  │  competitor.com                                   │
│  │        │  [Primary Competitor]                             │
│  └────────┘                                                    │
│                                                                │
│  Brief description of competitor...                           │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Quick Actions                                           │  │
│  │  [Ask Question] [View Report] [Set Alert] [Compare]    │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  ──────────────────────────────────────────────────────────── │
│  [Overview] [Product] [Pricing] [Team] [Funding] [News]      │
│  ──────────────────────────────────────────────────────────── │
│                                                                │
│  ┌──────────────────┐  ┌──────────────────┐  ┌─────────────┐ │
│  │  Funding         │  │  Team Size       │  │  Founded    │ │
│  │  $150M (Series C)│  │  250 employees   │  │  2019       │ │
│  │  ────────────    │  │  ────────────    │  │  ────────── │ │
│  │  +$50M (6mo ago) │  │  +20% YoY growth │  │  5 years    │ │
│  └──────────────────┘  └──────────────────┘  └─────────────┘ │
│                                                                │
│  Recent Activity                                              │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  📱 Jan 10  Launched mobile app v2.0                     │ │
│  │  💼 Jan 5   Hired VP of Sales (LinkedIn)                 │ │
│  │  📰 Dec 20  Featured in TechCrunch article               │ │
│  │  💰 Dec 15  Series C funding announced ($50M)            │ │
│  │  🎯 Dec 1   Updated pricing (15% increase)               │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  Product Features                           [View Comparison] │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  ✅ Analytics Dashboard                                   │ │
│  │  ✅ API Access                                            │ │
│  │  ✅ Integrations (20+)                                    │ │
│  │  ✅ Mobile Apps                                           │ │
│  │  ❌ Custom Branding                                       │ │
│  │  ❌ On-Premise Deployment                                 │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

---

### 4. Alerts Dashboard

**Layout:**

```
┌────────────────────────────────────────────────────────────────┐
│  Alerts                                    [Create Alert Rule] │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  [All] [Unread (8)] [High Priority] [This Week]              │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  🔴 HIGH    CompanyA raised $50M Series C                │ │
│  │  2 hours ago                              [Mark Read] [×] │ │
│  │                                                           │ │
│  │  CompanyA announced a $50M Series C funding round led by │ │
│  │  Sequoia Capital. This brings their total funding to...  │ │
│  │                                                           │ │
│  │  [View Details] [Ask Follow-up]                          │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  🟡 MEDIUM  CompanyB updated pricing                     │ │
│  │  5 hours ago                              [Mark Read] [×] │ │
│  │                                                           │ │
│  │  CompanyB increased their Professional tier from $149 to │ │
│  │  $199/month. This is their first price increase in...    │ │
│  │                                                           │ │
│  │  [View Details] [Ask Follow-up]                          │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │  🟢 LOW     CompanyC published blog post                 │ │
│  │  1 day ago                                [Mark Read] [×] │ │
│  │                                                           │ │
│  │  CompanyC published "The Future of Product Analytics"... │ │
│  │                                                           │ │
│  │  [View Details] [Ask Follow-up]                          │ │
│  └──────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

---

### 5. Compare View

**Layout:**

```
┌────────────────────────────────────────────────────────────────┐
│  Compare Competitors                                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Select competitors to compare: [CompanyA ▼] [CompanyB ▼]    │
│                                 [+ Add Competitor]             │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │              │ CompanyA      │ CompanyB      │ You       │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │ Pricing      │ $199/mo       │ $149/mo       │ $179/mo   │ │
│  │ (Pro tier)   │ ────────────  │ ────────────  │ ────────  │ │
│  │              │ 20% higher    │ ✅ Best value │ Middle    │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │ Features     │ 45 features   │ 38 features   │ 42        │ │
│  │              │ ────────────  │ ────────────  │ ────────  │ │
│  │              │ Most complete │ Basic         │ Competitive│ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │ Team Size    │ 250           │ 120           │ 180       │ │
│  │              │ ────────────  │ ────────────  │ ────────  │ │
│  │              │ Largest       │ Smallest      │ Medium    │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │ Funding      │ $150M         │ $30M          │ $75M      │ │
│  │              │ ────────────  │ ────────────  │ ────────  │ │
│  │              │ Well-funded   │ Capital light │ Moderate  │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │ G2 Rating    │ 4.5/5         │ 4.2/5         │ 4.3/5     │ │
│  │              │ ────────────  │ ────────────  │ ────────  │ │
│  │              │ (240 reviews) │ (85 reviews)  │ (120)     │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  [Export Comparison] [Share] [Save as Battle Card]           │
└────────────────────────────────────────────────────────────────┘
```

---

## Responsive Design

### Breakpoints

```css
/* Mobile: 320px - 640px */
@media (max-width: 640px) {
  /* Stack vertically, full-width cards */
}

/* Tablet: 641px - 1024px */
@media (min-width: 641px) and (max-width: 1024px) {
  /* 2-column layout, condensed sidebar */
}

/* Desktop: 1025px+ */
@media (min-width: 1025px) {
  /* Full layout with sidebar */
}
```

### Mobile Adaptations

1. **Bottom Navigation**: Replace sidebar with bottom tab bar
2. **Simplified Cards**: Show less metadata, tap to expand
3. **Query Input**: Full-screen modal for better focus
4. **Swipe Gestures**: Swipe to dismiss alerts, refresh feeds

---

## Interaction Patterns

### 1. Loading States

**Skeleton Screens** (preferred over spinners):

```jsx
<div className="skeleton-card">
  <div className="skeleton-header">
    <div className="skeleton-avatar"></div>
    <div>
      <div className="skeleton-line skeleton-line-short"></div>
      <div className="skeleton-line skeleton-line-shorter"></div>
    </div>
  </div>
  <div className="skeleton-body">
    <div className="skeleton-line"></div>
    <div className="skeleton-line"></div>
    <div className="skeleton-line skeleton-line-short"></div>
  </div>
</div>
```

**Progress Bars** (for multi-step processes):
```jsx
<div className="progress-bar">
  <div className="progress-fill" style={{ width: '60%' }}></div>
</div>
<p className="progress-label">Analyzing data... 60%</p>
```

---

### 2. Empty States

**No Competitors Yet:**

```
┌────────────────────────────────────┐
│                                    │
│          🔍                        │
│     No competitors yet             │
│                                    │
│  Start by adding your first        │
│  competitor to track their         │
│  activity and get insights.        │
│                                    │
│     [+ Add Competitor]             │
│                                    │
└────────────────────────────────────┘
```

**No Alerts:**

```
┌────────────────────────────────────┐
│                                    │
│          🔔                        │
│     All caught up!                 │
│                                    │
│  You have no new alerts.           │
│  We'll notify you when there's     │
│  important competitive activity.   │
│                                    │
│     [Create Alert Rule]            │
│                                    │
└────────────────────────────────────┘
```

---

### 3. Error States

**API Error:**

```jsx
<div className="error-state">
  <div className="error-icon">⚠️</div>
  <h3>Something went wrong</h3>
  <p>We couldn't load this data. Please try again.</p>
  <button className="btn btn-primary" onClick={retry}>
    Try Again
  </button>
  <p className="error-details">
    <a href="/support">Contact support</a> if this persists
  </p>
</div>
```

---

### 4. Animations

**Entrance Animations:**
- Cards: Fade in + slide up (150ms, ease-out)
- Modals: Scale from 95% to 100% + fade in (200ms)
- Alerts: Slide in from right (200ms, ease-out)

**Micro-interactions:**
- Button hover: Scale 1.02 + shadow increase
- Card hover: Shadow increase (smooth)
- Input focus: Border color transition + shadow

**Streaming Text:**
- Typewriter effect for agent responses (simulates thinking)
- Smooth scroll to new content as it appears

---

## Accessibility

### WCAG 2.1 AA Compliance

**Color Contrast:**
- Text: Minimum 4.5:1 contrast ratio
- Large text (18pt+): Minimum 3:1
- UI components: Minimum 3:1

**Keyboard Navigation:**
- All interactive elements accessible via Tab
- Logical tab order
- Visible focus indicators
- Escape to close modals/dropdowns

**Screen Reader Support:**
- Semantic HTML (nav, main, article, aside)
- ARIA labels where needed
- Alt text for all images
- Skip navigation links

**Example:**

```jsx
<button
  className="btn btn-primary"
  aria-label="Ask a competitive intelligence question"
>
  Ask Question
</button>

<img
  src="/logo.png"
  alt="CompanyA logo"
  role="img"
/>

<nav aria-label="Main navigation">
  <a href="#main-content" className="skip-link">
    Skip to main content
  </a>
  {/* navigation items */}
</nav>
```

---

## Design Tokens (for Developers)

Export design system as JSON for programmatic access:

```json
{
  "colors": {
    "primary": {
      "900": "#0A1628",
      "700": "#1E3A5F",
      "500": "#2E5C8A"
    }
  },
  "spacing": {
    "1": "0.25rem",
    "2": "0.5rem",
    "4": "1rem"
  },
  "typography": {
    "fontFamily": {
      "sans": "'Inter', sans-serif"
    },
    "fontSize": {
      "sm": "0.875rem",
      "base": "1rem",
      "lg": "1.125rem"
    }
  },
  "shadows": {
    "sm": "0 1px 2px 0 rgba(0, 0, 0, 0.05)",
    "md": "0 4px 6px -1px rgba(0, 0, 0, 0.1)"
  }
}
```

---

## Prototyping Tools

- **Design**: Figma (collaborative, developer handoff)
- **Prototyping**: Figma + ProtoPie (advanced interactions)
- **Design System**: Storybook (component documentation)
- **Usability Testing**: UserTesting.com, Hotjar

---

## Launch Checklist

### Before Launch
- [ ] All components in Storybook
- [ ] Design tokens exported
- [ ] Accessibility audit complete (WCAG AA)
- [ ] Cross-browser testing (Chrome, Firefox, Safari, Edge)
- [ ] Mobile responsive testing (iOS, Android)
- [ ] Loading states implemented
- [ ] Error states implemented
- [ ] Empty states implemented
- [ ] Keyboard navigation tested
- [ ] Screen reader testing (NVDA, JAWS, VoiceOver)

### Post-Launch
- [ ] Analytics tracking (button clicks, page views)
- [ ] Heatmaps (Hotjar)
- [ ] User testing (5-10 users per month)
- [ ] A/B testing critical flows
- [ ] Performance monitoring (Lighthouse scores)

---

**Document Version**: 1.0
**Last Updated**: 2026-01-12
**Figma File**: [Link to Figma design]
**Storybook**: [Link to Storybook instance]
