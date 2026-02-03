# {endpoint-name}

## Description
{Brief description of the endpoint - what it does, which agent handles it}

## DB Schema Dependencies

### Tables Referenced
- `{table_name}`: {Description of how this table is used}

### Fields Used
- `{table}.{field}`: {Type} - {Description}

## Parameters
```typescript
interface {EndpointName}Params {
  // Required parameters
  requiredParam: string;              // Description of param

  // Optional parameters
  optionalParam?: {
    nestedField: string;              // Description
  };

  // Common patterns
  language?: string;                  // "vi" | "en" - defaults to book.original_language
  options?: {
    skipStep?: boolean;               // Skip optional step (default: false)
  };
}
```

## Result
```typescript
interface {EndpointName}Result {
  // IDs
  entityId: string;                   // UUID of created/updated entity

  // Status
  status: "success" | "partial" | "failed";

  // Data
  data: {
    field: string;                    // Description
  };

  // Metadata
  meta?: {
    processingTime?: number;          // ms
    tokenUsage?: number;              // AI token count
  };
}
```

## Flow
```
1. Validate input parameters
   - Check required fields
   - Validate types and constraints

2. Fetch dependencies
   - Query {table} for {data}
   - Get related entities

3. Process
   - Step description
   - Call AI service (if applicable)

4. Save results
   - Update {table}.{field}
   - Create related records

5. Return result
```

## Error Handling
- **Validation Error**: Return 400 with field-specific errors
- **Not Found**: Return 404 if entity doesn't exist
- **AI Error**: Log and return partial result if possible
- **DB Error**: Rollback transaction, return 500

## Security Considerations (Optional)
- {Authentication requirements}
- {Authorization rules}
- {Rate limiting if applicable}

## Notes (Optional)
- {Important implementation notes}
- {Dependencies on other endpoints}
- {Performance considerations}
