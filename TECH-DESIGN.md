# [Project name]: Technical design document

- **Author:** [Name]
- **Reviewers:** [Names]
- **Status:** Draft | In review | Approved
- **Last updated:** [YYYY-MM-DD]
- **PRD:** [Link]

<!-- Guidance lives in HTML comments like this one. Most Markdown renderers
     hide them; delete them before you share the doc. -->

## Overview

<!-- One or two paragraphs: what is being built, the shape of the system,
     the constraints that drive the design, and the one thing it must get
     right. Link the PRD instead of restating requirements. -->

## Data flow

<!-- Follow one user action end to end, naming components as they appear.
     Then say what happens when a step fails: retries, idempotency and
     what the user sees. -->

1.
2.
3.

**On failure:**

## Components

<!-- One row per module or service with a clear owner. Responsibility =
     what it owns, what it takes in, what it produces. -->

| Component | Responsibility |
| --- | --- |
| [Name] | [Owns ... Inputs: ... Outputs: ...] |
| [Name] | |

## Data models

<!-- One block per entity: fields with types, then relationships written
     as sentences. Uniqueness and cardinality are business rules. -->

### [Entity]

| Field | Type | Notes |
| --- | --- | --- |
| id | uuid | Primary key |
| [field] | [type] | [Nullable? Why?] |
| created_at | timestamp | |

**Relationships**

- [Entity] belongs to [Other entity]
- [Entity] has many [Other entity]
- Unique ([field], [field])

## Schema

<!-- DDL for the core tables, with NOT NULL, CHECK, UNIQUE, foreign keys
     and the indexes your main queries need. Not on SQL? Use the
     TypeScript block instead. -->

```sql
CREATE TABLE [table_name] (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  [parent]_id uuid NOT NULL REFERENCES [parents](id),
  status text NOT NULL CHECK (status IN ('[a]', '[b]')),
  created_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX ON [table_name] ([parent]_id);
```

```ts
export type [Entity] = {
  id: string;
  [parent]Id: string;
  status: "[a]" | "[b]";
  createdAt: Date;
};
```

## Tech stack

<!-- A choice and a reason for every layer. "Not required" is a valid
     answer; say why, ideally by pointing at a PRD non-goal. -->

| Layer | Choice | Why |
| --- | --- | --- |
| Frontend | | |
| Backend | | |
| Database | | |
| Auth | | |
| LLM | | |
| Infra | | |
| Dev tools | | |

## Engineering tasks

<!-- In build order: foundations first (schema, access, deploy), then the
     riskiest component, then the rest. Each task says how you will know
     it is done. -->

1. **[Task title]:** [What to build]. Done when [observable condition].
2. **[Task title]:**
3. **[Task title]:**

## Open engineering questions

<!-- Unresolved decisions, plus every assumption this doc made that needs
     sign-off. Give each one an owner. -->

- [Question] Owner: [Name].
- [This doc assumed ... Needs sign-off from: Name.]
