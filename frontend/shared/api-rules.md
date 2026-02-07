# API Rules

> Universal API design and consumption rules for Claude Code. Apply to all projects regardless of tech stack.

## API Service Layer

- All API calls must go through a **centralized service layer** — never call `fetch` or `axios` directly from components:

```
src/
└── services/
    ├── api.ts              ← Base HTTP client with interceptors
    ├── auth.service.ts     ← Auth-related API calls
    ├── user.service.ts     ← User-related API calls
    └── index.ts            ← Re-exports
```

```ts
// ❌ Incorrect — direct API call in component
const response = await axios.get("/api/users");

// ✅ Correct — through service layer
import { userService } from "@/services";
const users = await userService.getAll();
```

## Base HTTP Client

- Create a single base HTTP client with configured interceptors:

```ts
// services/api.ts
import axios from "axios";

const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 10000,
  headers: { "Content-Type": "application/json" },
});

// Request interceptor — attach auth token
api.interceptors.request.use((config) => {
  const token = getToken();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Response interceptor — handle errors globally
api.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      await refreshToken();
      return api.request(error.config);
    }
    return Promise.reject(error);
  }
);

export default api;
```

## Error Handling

- **Every API call** must have error handling. Never leave API calls without try-catch or `.catch()`:

```ts
// ❌ Incorrect — no error handling
const fetchUsers = async () => {
  const { data } = await api.get("/users");
  return data;
};

// ✅ Correct — proper error handling
const fetchUsers = async (): Promise<User[]> => {
  try {
    const { data } = await api.get<User[]>("/users");
    return data;
  } catch (error) {
    console.error("Failed to fetch users:", error);
    throw error;
  }
};
```

- Show user-friendly error messages — never display raw API errors to users.
- Implement global error handling for common HTTP status codes (401, 403, 404, 500).

## Request/Response Typing

- Type all API requests and responses — never use `any`:

```ts
// ❌ Incorrect
const getUser = async (id: number): Promise<any> => { ... };

// ✅ Correct
interface UserResponse {
  id: number;
  name: string;
  email: string;
}

const getUser = async (id: number): Promise<UserResponse> => { ... };
```

## Loading and Error States

- Every API call in the UI must have three states: **loading**, **success**, and **error**:

```ts
const isLoading = ref(false);
const error = ref<string | null>(null);
const data = ref<User[]>([]);

const fetchUsers = async () => {
  isLoading.value = true;
  error.value = null;
  try {
    data.value = await userService.getAll();
  } catch (e) {
    error.value = "Failed to load users. Please try again.";
  } finally {
    isLoading.value = false;
  }
};
```

- Show loading indicators (spinners, skeletons) during API calls.
- Show error states with retry options.
- Show empty states when data is loaded but empty.

## API Naming Conventions

```ts
// Service method naming
userService.getAll()           // GET /users
userService.getById(id)        // GET /users/:id
userService.create(data)       // POST /users
userService.update(id, data)   // PUT /users/:id
userService.patch(id, data)    // PATCH /users/:id
userService.delete(id)         // DELETE /users/:id
userService.search(params)     // GET /users/search?q=...
```

## Request Cancellation

- Cancel pending requests when the component unmounts or when a new request supersedes the previous one:

```ts
const controller = new AbortController();

const fetchData = async () => {
  const { data } = await api.get("/data", { signal: controller.signal });
  return data;
};

// On unmount or new request
controller.abort();
```

## Rules

- Never expose API keys or secrets in client-side code.
- Use environment variables for all API base URLs and configuration.
- Implement request retry logic for transient failures (network errors, 503).
- Rate limit client-side requests — debounce search inputs, throttle rapid actions.
- Log API errors for monitoring and debugging — but never log sensitive data (tokens, passwords).
- Use appropriate HTTP methods — GET for reads, POST for creates, PUT/PATCH for updates, DELETE for deletes.
