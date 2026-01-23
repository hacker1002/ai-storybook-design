# {Feature Name}

## Description
{Brief description of the feature - what it does, user goals, AI involvement}

## Flow Overview
```
┌─────────────────┐
│   Phase 1       │ ←→ {description}
│   {name}        │
└────────┬────────┘
         │ {transition condition}
         ▼
┌─────────────────┐
│   Phase 2       │
│   {name}        │
└────────┬────────┘
         │ {transition condition}
         ▼
┌─────────────────┐
│   Phase N       │
│   {name}        │
└─────────────────┘
```

---

## Phase 1: {Name}

### Description
{Detailed description of this phase}

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
- {Alternative exit condition}

---

## Phase 2: {Name}

### Description
{Detailed description of this phase}

### User Actions
| Action | Description |
|--------|-------------|
| `{action}` | {what happens} |

### State Management
```typescript
interface Phase2State {
  status: "processing" | "completed";
  // ...
}
```

---

## UI Components

### {ComponentName}
```typescript
interface {ComponentName}Props {
  prop: string;
  onAction: () => void;
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

## Error Handling

| Error | Action |
|-------|--------|
| {error type} | {how to handle} |

---

## Edge Cases

### 1. {Edge case name}
- **Trigger**: {what causes this}
- **Solution**: {how to handle}

---

## Dependencies

### Supabase Queries
```typescript
supabase.from('{table}').select('{fields}')
```

### API Endpoints
- `{METHOD} {path}` - {description}

### DB Tables
- `{table}`: {how it's used}
