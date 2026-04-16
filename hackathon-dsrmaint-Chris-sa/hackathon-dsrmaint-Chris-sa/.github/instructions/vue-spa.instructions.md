---
description: 'Vue.js SPA conventions for Shamrock Foods applications'
applyTo: '**/*.vue, **/*.ts, **/*.js'
---

# Shamrock Foods Vue.js SPA Conventions

## Architecture Conventions

- Favor the **Composition API** over the Options API
- Use `<script setup>` syntax (with `lang="ts"` for new applications)
- Organize components and composables by feature or domain
- Extract reusable logic into composable functions in a `composables/` directory
- Use **Pinia** for global state — structure stores by domain with actions, state, and getters
- Separate API calls into service files — do not call APIs directly from components or stores

## Component Library & MCP

- Use `@sfc-enterprise-ui/one-erp-components-ts` for all UI components
- **For component selection, props, slots, and usage examples: query the `@oneErp-components` MCP server**
- Use `SvgConstants` from the library when referencing SVG names
- Use `OneErpColors` with the `useTheme` composable for all color values
- Use `@sfc-enterprise-ui/one-erp-theme` for device breakpoints and shared styles

## Component Pattern (§8.3)

- **New applications:** Use TypeScript with `<script setup lang="ts">`
- **Existing applications:** May use plain JavaScript with `setup()`
- Functions that are `await`ed use the `Async` suffix (e.g., `handleSubmitAsync`, `loadOrderAsync`)

## Enterprise Fetch API (§8.1)

- **Always** use `@sfc-enterprise-ui/fetch-api` — never raw `fetch()` or `axios`
- Available methods: `getAsync`, `postAsync`, `putAsync`, `deleteAsync`
- Error banners display automatically for 400/500 responses — no manual error UI needed
- Auth token injection is centralized in the package
- The library is plain JavaScript — do **not** use generic type parameters (e.g., `getAsync<T>()` is wrong)
- The library auto-unwraps and parses response bodies — do **not** call `.json()` on results
- Provide **only** relative endpoint paths — the library prepends the base URL automatically

```javascript
import { getAsync, postAsync } from '@sfc-enterprise-ui/fetch-api';

// ✅ CORRECT — relative path, no .json(), no generics
const customers = await postAsync('/api/customers/search', filter);

// ❌ WRONG — manual base URL
const customers = await postAsync('https://localhost:7230/api/customers/search', filter);
```

## TypeScript Strict Mode Patterns

When using TypeScript with Vue 3 Composition API in strict mode:

- **Let TypeScript infer** for primitives: `ref(0)`, `ref('')`, `ref(false)`
- **Use type assertions** for complex types: `ref([]) as { value: Customer[] }`
- **Use explicit types** for unions: `ref<'active' | 'inactive'>('active')`, `ref<User | null>(null)`
- **Use `defineProps`** with type parameter for typed props:
  ```typescript
  const props = defineProps<{ items: Item[]; loading?: boolean }>();
  ```
- Review this pattern when `@sfc-enterprise-ui/one-erp-components-ts` ships full `.d.ts` definitions

## Vue Router Conventions

- **`path`** — ALWAYS starts with `/`: `path: '/customers'`
- **`name`** — NEVER starts with `/`: `name: 'customers'`

```typescript
const routes: RouteRecordRaw[] = [
  { path: '/', name: 'home', components: { content: HomePage } },
  { path: '/customers', name: 'customers', components: { content: CustomerSearch } }
];
```

## Search Form Pattern

- Use **local `ref` values** for form inputs — do not bind directly to search state
- Apply local values to search state only when the user presses **Enter** or clicks **Search**
- This prevents excessive API calls, mid-typing validation errors, and unnecessary re-renders
- **Clear** button resets both local inputs and search results
- Do **not** use this pattern for autocomplete/typeahead fields that require real-time results

## Styling

- Use `<style scoped>` for component-level styles
- Follow BEM or functional CSS conventions for class naming
- Use CSS Grid and Flexbox for responsive, mobile-first layouts

## Timezone Handling (§12.5)

- Use the **`dayjs`** npm package for date parsing, formatting, and timezone manipulation
- API returns all dates in **UTC** — browser converts to local time for display
- When sending `DateTime` property types to the API, convert local time to UTC using `dayjs`
- `new Date(utcString)` automatically converts to browser local time for display

```javascript
// Sending to API — convert local to UTC
const deliveryDateUtc = dayjs(request.deliveryDate).utc().format();

// Displaying from API — UTC auto-converts to local
const deliveryDate = new Date(order.deliveryDate);
console.log(deliveryDate.toLocaleString()); // local time
```

## Formatting & Linting

### Prettier Configuration

```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "none",
  "endOfLine": "auto"
}
```

- **Semicolons**: Required
- **Quotes**: Single quotes for JS/TS, double quotes for HTML attributes in Vue templates
- **Indentation**: 2 spaces
- **Trailing commas**: Not allowed

### ESLint Configuration

- **Parser**: `vue-eslint-parser` with `@typescript-eslint/parser`
- **Plugins**: `eslint-plugin-vue` (`flat/recommended`) + `eslint-plugin-prettier/recommended`
- **Coverage**: `**/*.vue`, `**/*.ts`, `**/*.tsx`
- **Verify**: `npx eslint . --ext .vue,.ts,.tsx --fix`

## Cross-References

- **API Gateway patterns:** The Vue SPA calls API Gateway endpoints (not microservices directly). API Gateways unwrap `BaseResult<T>` to standard HTTP status codes via `GetResult()`. See `csharp-api-gateway` instructions for the BFF pattern.
