# {Feature Name}

## Description
{Brief description of the feature - what it does, user goals, AI involvement}

## Flow Overview
```
┌─────────────────────────────────────┐
│   Phase 1: {name}                   │  {description}
│   [CLIENT]                          │
└────────────────┬────────────────────┘
                 │ {transition condition}
                 ▼
┌─────────────────────────────────────┐
│   Phase 2: {name}                   │  {description}
│   [CLIENT → DB]                     │
└────────────────┬────────────────────┘
                 │ {transition condition}
                 ▼
┌─────────────────────────────────────┐
│   Phase 3: {name}                   │  {description}
│   [CLIENT → API → AI]               │
└────────────────┬────────────────────┘
                 │ {transition condition}
                 ▼
┌─────────────────────────────────────┐
│   Phase N: {name}                   │  {description}
│   [API BACKGROUND]                  │
└─────────────────────────────────────┘
```

---

## Phase 1: {Name}

### Flow Pattern
`[CLIENT]`

### Description
{Detailed description of this phase}

### Responsibilities
**CLIENT:**
- Validate required fields before calling DB/API
- Display form/UI to collect missing info
- Local state management

### User Actions
| Action | Description |
|--------|-------------|
| `{action}` | {what happens} |

### State Management
```typescript
interface Phase1State {
  status: "active" | "completed";
  data: {
    field: string;
  };
}
```

### Exit Conditions
- {Condition to move to next phase}

---

## Phase 2: {Name}

### Flow Pattern
`[CLIENT → DB]`

### Description
{Detailed description of this phase - simple CRUD via Supabase client}

### Responsibilities
**CLIENT:**
- Build query params
- Handle loading/error states
- Display data from DB

### User Actions
| Action | Description |
|--------|-------------|
| `{action}` | {what happens} |

### State Management
```typescript
interface Phase2State {
  status: "loading" | "success" | "error";
  data: any[];
}
```

---

## Phase 3: {Name}

### Flow Pattern
`[CLIENT → API → AI]`

### Description
{Detailed description of this phase - AI generation flow}

### Responsibilities
**CLIENT:**
- Combine all params into request payload
- Handle loading/error/streaming states
- Parse and display AI response

**API:**
- Get extra context from DB (story settings, related entities)
- Build prompt from template + context
- Send to AI Provider
- Parse AI response, validate structure
- Return formatted response to client

**AI:**
- Generate content based on prompt

### User Actions
| Action | Description |
|--------|-------------|
| `{action}` | {what happens} |

### State Management
```typescript
interface Phase3State {
  status: "idle" | "generating" | "completed" | "error";
  result?: any;
}
```

---

## Phase N: {Name} (Optional - Background Tasks)

### Flow Pattern
`[API BACKGROUND]`

### Description
{Detailed description of background processing phase}

### Responsibilities
**CLIENT:**
- Trigger API to start job
- Poll for job status OR listen to realtime updates
- Display progress indicator
- Handle completion/failure states

**API:**
- Validate request
- Trigger background job/queue
- Return job ID to client immediately

**API (Background Worker):**
- Execute long-running tasks (image generation, export, etc.)
- Update job status in DB
- Notify client when complete

### State Management
```typescript
interface PhaseNState {
  jobId: string;
  status: "queued" | "processing" | "completed" | "failed";
  progress?: number;
  result?: any;
  error?: string;
}
```

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/{endpoint}` | {description} |

### Request/Response
```typescript
// Request
interface RequestBody {
  field: string;
}

// Response
interface ResponseBody {
  status: "success" | "error";
  data: any;
}
```

---

## DB Queries (Supabase)

| Operation | Table | Description |
|-----------|-------|-------------|
| SELECT | `{table}` | {description} |
| INSERT | `{table}` | {description} |

---

## Error Handling

| Layer | Error | Action |
|-------|-------|--------|
| CLIENT | {error type} | {how to handle} |
| API | {error type} | {how to handle} |
| DB | {error type} | {how to handle} |

---

## Edge Cases

### 1. {Edge case name}
- **Trigger**: {what causes this}
- **Layer**: {which layer handles}
- **Solution**: {how to handle}

