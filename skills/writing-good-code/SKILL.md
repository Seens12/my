---
name: writing-good-code
description: Writing readable, maintainable code. Activated when applying naming conventions, function decomposition, or Early Return patterns.
metadata:
  mcpmarket-version: 1.0.0
---
# Writing Good Code

> Readable, easy-to-change code

## Core Principles

### 1. Names Are Documentation

```typescript
// AVOID
const d = new Date().getTime() - startTime;
if (d > 3000) { ... }

// PREFER
const elapsedMs = new Date().getTime() - startTime;
const TIMEOUT_MS = 3000;
if (elapsedMs > TIMEOUT_MS) { ... }
```

### 2. One Function, One Job

```typescript
// AVOID: multiple responsibilities
async function handleSubmit(data) {
  const validated = schema.parse(data);
  const response = await api.post('/users', validated);
  analytics.track('user_created');
  toast.success('Done');
  router.push('/dashboard');
  return response;
}

// PREFER: separate concerns
async function createUser(data) {
  return api.post('/users', schema.parse(data));
}

function handleSubmit(data) {
  createUser(data)
    .then(() => {
      analytics.track('user_created');
      toast.success('Done');
      router.push('/dashboard');
    });
}
```

### 3. Name Your Conditions

```typescript
// AVOID
if (user.age >= 19 && user.membership === 'premium' && !user.isBanned) {
  showContent();
}

// PREFER
const canAccessPremiumContent =
  user.age >= 19 &&
  user.membership === 'premium' &&
  !user.isBanned;

if (canAccessPremiumContent) {
  showContent();
}
```

### 4. Early Return to Reduce Nesting

```typescript
// AVOID
function getDiscount(user) {
  if (user) {
    if (user.membership === 'premium') {
      if (user.years > 2) {
        return 0.2;
      } else {
        return 0.1;
      }
    } else {
      return 0;
    }
  }
  return 0;
}

// PREFER
function getDiscount(user) {
  if (!user) return 0;
  if (user.membership !== 'premium') return 0;
  if (user.years > 2) return 0.2;
  return 0.1;
}
```

### 5. Keep Related Code Together

```typescript
// AVOID: type, constants, utils scattered across folders
import { User } from '@/types/user';
import { USER_STATUS } from '@/constants/user';
import { formatUserName } from '@/utils/user';

// PREFER: co-located code
// features/user/index.ts
export interface User { ... }
export const USER_STATUS = { ... };
export function formatUserName(user: User) { ... }
```

## Component Patterns

### Props: Only What You Need

```typescript
// AVOID: passing entire object
function UserAvatar({ user }: { user: User }) {
  return <img src={user.avatar} alt={user.name} />;
}

// PREFER: only needed fields
function UserAvatar({ src, alt }: { src: string; alt: string }) {
  return <img src={src} alt={alt} />;
}
```

### Conditional Rendering

```typescript
// AVOID: nested ternaries
{isLoading ? <Spinner /> : error ? <Error /> : data ? <Content /> : null}

// PREFER: early return
function Component() {
  if (isLoading) return <Spinner />;
  if (error) return <Error />;
  if (!data) return null;
  return <Content data={data} />;
}
```

### Extract Logic into Custom Hooks

```typescript
// PREFER: hooks for logic, components for UI
function useSearch() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 300);

  const { data } = useQuery({
    queryKey: ['search', debouncedQuery],
    queryFn: () => search(debouncedQuery),
    enabled: debouncedQuery.length > 0,
  });

  return { query, setQuery, results: data };
}

function SearchResults() {
  const { query, setQuery, results } = useSearch();
  return (/* UI */);
}
```

## DO NOT

```typescript
// AVOID: derived state as separate useState
const [items, setItems] = useState([]);
const [total, setTotal] = useState(0);
useEffect(() => {
  setTotal(items.reduce((sum, item) => sum + item.price, 0));
}, [items]);
// PREFER: compute directly
const total = items.reduce((sum, item) => sum + item.price, 0);

// AVOID: hidden side effects
function formatPrice(price: number) {
  analytics.track('price_viewed'); // unexpected!
  return `$${price.toFixed(2)}`;
}

// AVOID: premature abstraction
function GenericModal<T>({ data, renderHeader, renderBody, ... }: GenericModalProps<T>) { ... }
// PREFER: start concrete, abstract when needed
function ConfirmDeleteModal({ onConfirm, onCancel }) { ... }
```

|Principle|Description|
|---|---|
|Read > Write|Code is read far more often than written|
|Understand > Work|"Understandable code" beats "working code"|
|YAGNI|Don't anticipate future requirements, write what's needed now|
