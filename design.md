# Design Document: h2s-frontend

## Overview

The h2s-frontend is a React-based single page application (SPA) that provides an intuitive interface for users to interact with an AI-powered learning assistant. The application follows a modern component-based architecture with centralized state management, clean separation of concerns, and a focus on user experience.

The frontend integrates with an existing Node.js/Express backend that provides authentication services and AI assistant communication. The design emphasizes simplicity, responsiveness, and maintainability suitable for a hackathon demonstration while maintaining production-quality code structure.

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    React Application                     │
│  ┌────────────────────────────────────────────────────┐ │
│  │              Routing Layer (React Router)          │ │
│  │  - Public Routes (Login, Register)                 │ │
│  │  - Protected Routes (Chat)                         │ │
│  └────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────┐ │
│  │           State Management (Context API)           │ │
│  │  - AuthContext (user, token, login, logout)        │ │
│  │  - ChatContext (messages, sendMessage, loading)    │ │
│  └────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────┐ │
│  │                 Component Layer                     │ │
│  │  - Auth Components (Login, Register)               │ │
│  │  - Chat Components (ChatInterface, MessageList)    │ │
│  │  - Shared Components (Button, Input, ErrorMsg)     │ │
│  └────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────┐ │
│  │              API Integration Layer                  │ │
│  │  - API Client (axios instance with interceptors)   │ │
│  │  - Auth Service (register, login)                  │ │
│  │  - Assistant Service (sendMessage)                 │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
                          │
                          │ HTTP/REST
                          ▼
┌─────────────────────────────────────────────────────────┐
│              Node.js/Express Backend API                 │
│  - POST /auth/register                                   │
│  - POST /auth/login                                      │
│  - POST /assistant/ask (requires JWT)                    │
└─────────────────────────────────────────────────────────┘
```

### Technology Stack

- **Framework**: React 18+ with functional components and hooks
- **Routing**: React Router v6 for client-side routing
- **State Management**: React Context API with useContext and useReducer
- **HTTP Client**: Axios for API communication
- **Styling**: CSS Modules or Tailwind CSS for component styling
- **Build Tool**: Vite for fast development and optimized production builds
- **Storage**: Browser localStorage for JWT token persistence

### Design Principles

1. **Component Composition**: Small, reusable components with single responsibilities
2. **Separation of Concerns**: Clear boundaries between UI, state, and API logic
3. **Declarative UI**: React's declarative approach for predictable rendering
4. **Responsive Design**: Mobile-first approach with flexible layouts
5. **Error Boundaries**: Graceful error handling at component and API levels

## Components and Interfaces

### Core Components

#### 1. App Component
```typescript
// Root component that sets up routing and global providers
interface AppProps {}

Component Structure:
- AuthProvider wrapper
- ChatProvider wrapper
- Router with route definitions
- Global error boundary
```

#### 2. AuthContext Provider
```typescript
interface AuthContextValue {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  login: (email: string, password: string) => Promise<void>;
  register: (email: string, password: string) => Promise<void>;
  logout: () => void;
  loading: boolean;
  error: string | null;
}

interface User {
  id: number;
  email: string;
}
```

#### 3. ChatContext Provider
```typescript
interface ChatContextValue {
  messages: Message[];
  sendMessage: (text: string) => Promise<void>;
  loading: boolean;
  error: string | null;
  clearError: () => void;
}

interface Message {
  id: string;
  text: string;
  sender: 'user' | 'ai';
  timestamp: Date;
}
```

#### 4. ProtectedRoute Component
```typescript
interface ProtectedRouteProps {
  children: React.ReactNode;
}

// Redirects to /login if not authenticated
// Renders children if authenticated
```

#### 5. Login Component
```typescript
interface LoginProps {}

State:
- email: string
- password: string
- errors: { email?: string; password?: string }

Handlers:
- handleSubmit: validates and calls authContext.login
- handleInputChange: updates form state
```

#### 6. Register Component
```typescript
interface RegisterProps {}

State:
- email: string
- password: string
- confirmPassword: string
- errors: { email?: string; password?: string; confirmPassword?: string }

Handlers:
- handleSubmit: validates and calls authContext.register
- handleInputChange: updates form state
```

#### 7. ChatInterface Component
```typescript
interface ChatInterfaceProps {}

State:
- inputValue: string

Handlers:
- handleSendMessage: sends message via chatContext
- handleInputChange: updates input state
- handleKeyPress: sends on Enter key

Child Components:
- MessageList
- ChatInput
- LoadingIndicator
```

#### 8. MessageList Component
```typescript
interface MessageListProps {
  messages: Message[];
}

Features:
- Auto-scroll to bottom on new messages
- Visual distinction between user and AI messages
- Empty state for no messages
```

#### 9. MessageBubble Component
```typescript
interface MessageBubbleProps {
  message: Message;
}

Styling:
- Different colors for user vs AI
- Timestamp display
- Proper text formatting
```

#### 10. ChatInput Component
```typescript
interface ChatInputProps {
  value: string;
  onChange: (value: string) => void;
  onSend: () => void;
  disabled: boolean;
}

Features:
- Text input field
- Send button
- Disabled state during loading
- Enter key support
```

### API Service Layer

#### API Client Configuration
```typescript
// axios instance with base configuration
const apiClient = axios.create({
  baseURL: 'http://localhost:3000',
  headers: {
    'Content-Type': 'application/json'
  }
});

// Request interceptor to add JWT token
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Response interceptor for error handling
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Clear token and redirect to login
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);
```

#### Auth Service
```typescript
interface RegisterRequest {
  email: string;
  password: string;
}

interface RegisterResponse {
  message: string;
}

interface LoginRequest {
  email: string;
  password: string;
}

interface LoginResponse {
  token: string;
}

const authService = {
  register: (data: RegisterRequest): Promise<RegisterResponse> => 
    apiClient.post('/auth/register', data),
  
  login: (data: LoginRequest): Promise<LoginResponse> => 
    apiClient.post('/auth/login', data)
};
```

#### Assistant Service
```typescript
interface AskRequest {
  message: string;
}

interface AskResponse {
  response: string;
}

const assistantService = {
  sendMessage: (data: AskRequest): Promise<AskResponse> => 
    apiClient.post('/assistant/ask', data)
};
```

## Data Models

### Frontend Data Models

#### User Model
```typescript
interface User {
  id: number;
  email: string;
}
```

#### Message Model
```typescript
interface Message {
  id: string;           // UUID generated on frontend
  text: string;         // Message content
  sender: 'user' | 'ai'; // Message sender type
  timestamp: Date;      // When message was created
}
```

#### Auth State Model
```typescript
interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  loading: boolean;
  error: string | null;
}
```

#### Chat State Model
```typescript
interface ChatState {
  messages: Message[];
  loading: boolean;
  error: string | null;
}
```

### API Request/Response Models

#### Registration
```typescript
// Request
POST /auth/register
{
  "email": "user@example.com",
  "password": "password123"
}

// Success Response (200)
{
  "message": "User registered successfully"
}

// Error Response (400/500)
{
  "error": "Missing fields" | "DB Error" | "Server error"
}
```

#### Login
```typescript
// Request
POST /auth/login
{
  "email": "user@example.com",
  "password": "password123"
}

// Success Response (200)
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}

// Error Response (401/500)
{
  "error": "Invalid credentials" | "DB Error"
}
```

#### Send Message
```typescript
// Request
POST /assistant/ask
Headers: { "Authorization": "Bearer <token>" }
{
  "message": "How do I learn React?"
}

// Success Response (200)
{
  "response": "React is a JavaScript library for building user interfaces..."
}

// Error Response (500)
{
  "error": "AI dynamic handling failed"
}
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*


### Property 1: Valid registration triggers API call
*For any* valid email and password combination, submitting the registration form should send a POST request to /auth/register with the provided credentials.
**Validates: Requirements 1.1**

### Property 2: Successful registration shows success and redirects
*For any* successful registration API response, the application should display a success message and navigate to the login page.
**Validates: Requirements 1.2**

### Property 3: Email validation rejects invalid formats
*For any* string that does not match valid email format (missing @, invalid domain, etc.), the registration form should prevent submission and display a validation error.
**Validates: Requirements 1.4**

### Property 4: Password validation enforces minimum length
*For any* password string with fewer than 6 characters, the registration form should prevent submission and display a validation error.
**Validates: Requirements 1.5**

### Property 5: Valid login triggers API call
*For any* valid email and password combination, submitting the login form should send a POST request to /auth/login with the provided credentials.
**Validates: Requirements 2.1**

### Property 6: Successful login stores token
*For any* successful login response containing a JWT token, the application should store that token in localStorage under the key 'token'.
**Validates: Requirements 2.2**

### Property 7: Successful login redirects to chat
*For any* successful login, the application should navigate the user to the chat interface route.
**Validates: Requirements 2.3**

### Property 8: API errors display error messages
*For any* API error response (registration failure, login failure, message send failure, network error), the application should extract and display an appropriate error message to the user.
**Validates: Requirements 1.3, 2.4, 8.1, 8.3**

### Property 9: Token persistence enables auto-authentication
*For any* valid JWT token stored in localStorage, when the application loads, it should automatically set the authenticated state to true and populate the user context.
**Validates: Requirements 2.5, 3.1, 3.2**

### Property 10: Logout clears token and redirects
*For any* authenticated session, when the user logs out, the application should remove the JWT token from localStorage and navigate to the login page.
**Validates: Requirements 3.3, 3.4**

### Property 11: 401 responses trigger re-authentication
*For any* API request that returns a 401 Unauthorized status, the application should clear the stored JWT token and redirect to the login page.
**Validates: Requirements 3.5**

### Property 12: Unauthenticated users cannot access protected routes
*For any* protected route, when an unauthenticated user (no valid token) attempts to access it, the application should redirect to the login page instead of rendering the protected component.
**Validates: Requirements 4.1, 4.4**

### Property 13: Authenticated users can access protected routes
*For any* protected route, when an authenticated user (valid token exists) attempts to access it, the application should render the requested component.
**Validates: Requirements 4.2**

### Property 14: Authenticated requests include token header
*For any* API request made while a JWT token exists in localStorage, the request should include an Authorization header with the format "Bearer <token>".
**Validates: Requirements 4.3**

### Property 15: Message submission triggers API call
*For any* non-empty message text, when the user submits it, the application should send a POST request to /assistant/ask with the message in the request body.
**Validates: Requirements 5.1**

### Property 16: User messages display immediately (optimistic update)
*For any* message submitted by the user, the message should appear in the chat interface immediately, before the AI response is received.
**Validates: Requirements 5.2**

### Property 17: AI responses display when received
*For any* successful response from the /assistant/ask endpoint, the AI's response text should be displayed as a new message in the chat interface.
**Validates: Requirements 5.3**

### Property 18: Loading indicator shows during message processing
*For any* message submission, while waiting for the API response, the application should display a loading indicator.
**Validates: Requirements 5.4**

### Property 19: Failed messages show error and allow retry
*For any* message that fails to send (network error or API error), the application should display an error message and keep the input field populated to allow the user to retry.
**Validates: Requirements 5.5**

### Property 20: All session messages are displayed
*For any* set of messages in the chat state, all messages should be rendered in the message list component.
**Validates: Requirements 6.1**

### Property 21: New messages trigger auto-scroll
*For any* new message added to the chat, the message list should automatically scroll to show the latest message at the bottom.
**Validates: Requirements 6.2**

### Property 22: User and AI messages are visually distinct
*For any* message in the chat interface, user messages and AI messages should have different visual styling (colors, alignment, or other visual indicators).
**Validates: Requirements 6.3**

### Property 23: Messages display in chronological order
*For any* set of messages with timestamps, the messages should be displayed sorted by timestamp in ascending order (oldest first).
**Validates: Requirements 6.4**

### Property 24: Session state persists until page refresh
*For any* messages added to the chat state during a session, those messages should remain in state and be displayed until the page is refreshed or the user logs out.
**Validates: Requirements 9.2**

### Property 25: Form validation highlights invalid fields
*For any* form with validation errors, the invalid fields should be visually highlighted and display specific error messages indicating what needs to be corrected.
**Validates: Requirements 8.4**

### Property 26: Error messages clear on corrective action
*For any* displayed error message, when the user takes action to correct the error (e.g., typing in a field, resubmitting), the error message should be cleared.
**Validates: Requirements 8.5**

### Property 27: JSON responses parse correctly
*For any* API response with Content-Type application/json, the response body should be parsed as JSON and made available to the application.
**Validates: Requirements 10.3**

### Property 28: HTTP status codes trigger appropriate handling
*For any* API response, the application should handle different HTTP status codes appropriately: 200 (success), 401 (unauthorized - clear token and redirect), 400 (client error - show error), 500 (server error - show error).
**Validates: Requirements 10.4**

## Error Handling

### Error Categories and Handling Strategies

#### 1. Network Errors
**Scenario**: Backend API is unreachable, network connection lost, timeout

**Handling**:
- Catch network errors in API client interceptor
- Display user-friendly message: "Unable to connect to the server. Please check your internet connection."
- Provide retry mechanism for failed requests
- Maintain application state (don't lose user input)

#### 2. Authentication Errors (401)
**Scenario**: Invalid or expired JWT token, unauthorized access

**Handling**:
- Intercept 401 responses globally in axios interceptor
- Clear stored JWT token from localStorage
- Reset authentication state to unauthenticated
- Redirect user to login page
- Display message: "Your session has expired. Please log in again."

#### 3. Validation Errors (400)
**Scenario**: Invalid input data, missing required fields

**Handling**:
- Display specific validation errors next to relevant form fields
- Prevent form submission until validation passes
- Client-side validation before API calls (email format, password length)
- Server-side validation errors displayed from API response

#### 4. Server Errors (500)
**Scenario**: Backend crashes, database errors, AI service unavailable

**Handling**:
- Display generic error message: "Something went wrong. Please try again later."
- Log error details to console for debugging
- Provide retry option for user
- Don't expose internal error details to user

#### 5. Component Errors
**Scenario**: React component rendering errors, unexpected state

**Handling**:
- Implement Error Boundary component at app root
- Catch and log component errors
- Display fallback UI: "Something went wrong. Please refresh the page."
- Prevent entire app crash from single component failure

### Error State Management

```typescript
interface ErrorState {
  message: string;
  type: 'network' | 'auth' | 'validation' | 'server' | 'component';
  field?: string; // For validation errors
  retryable: boolean;
}

// Error handling in contexts
const handleError = (error: AxiosError): ErrorState => {
  if (!error.response) {
    return {
      message: 'Unable to connect to the server',
      type: 'network',
      retryable: true
    };
  }
  
  switch (error.response.status) {
    case 401:
      return {
        message: 'Session expired. Please log in again.',
        type: 'auth',
        retryable: false
      };
    case 400:
      return {
        message: error.response.data.error || 'Invalid input',
        type: 'validation',
        retryable: false
      };
    case 500:
      return {
        message: 'Server error. Please try again later.',
        type: 'server',
        retryable: true
      };
    default:
      return {
        message: 'An unexpected error occurred',
        type: 'server',
        retryable: true
      };
  }
};
```

## Testing Strategy

### Overview

The testing strategy employs a dual approach combining unit tests for specific scenarios and property-based tests for universal correctness properties. This ensures both concrete bug detection and comprehensive coverage of input spaces.

### Unit Testing

Unit tests focus on:
- **Specific examples**: Concrete test cases that demonstrate correct behavior
- **Edge cases**: Boundary conditions like empty inputs, maximum lengths, special characters
- **Integration points**: Component interactions, context providers, API service integration
- **Error conditions**: Specific error scenarios and recovery paths

**Testing Library**: React Testing Library + Jest
- Component rendering and user interactions
- Mock API calls with MSW (Mock Service Worker)
- Test user flows (registration, login, sending messages)

**Example Unit Tests**:
```typescript
// Login component
- Should render email and password fields
- Should show validation error for invalid email
- Should call login API with correct credentials
- Should redirect to chat on successful login
- Should display error message on login failure

// ChatInterface component
- Should display welcome message when no messages exist
- Should render all messages from state
- Should disable input during message sending
- Should show loading indicator while waiting for response
- Should handle empty message submission

// ProtectedRoute component
- Should redirect to login when not authenticated
- Should render children when authenticated
- Should check authentication before rendering
```

### Property-Based Testing

Property tests verify universal properties across randomized inputs to catch edge cases that unit tests might miss.

**Testing Library**: fast-check (JavaScript property-based testing library)

**Configuration**:
- Minimum 100 iterations per property test
- Each test tagged with feature name and property number
- Tag format: `// Feature: h2s-frontend, Property N: <property description>`

**Property Test Implementation**:

Each correctness property from the design document should be implemented as a property-based test. The tests should:
1. Generate random valid inputs using fast-check arbitraries
2. Execute the system behavior
3. Assert the property holds for all generated inputs

**Example Property Tests**:

```typescript
// Property 3: Email validation rejects invalid formats
// Feature: h2s-frontend, Property 3: Email validation rejects invalid formats
test('email validation rejects all invalid formats', () => {
  fc.assert(
    fc.property(
      fc.string().filter(s => !isValidEmail(s)), // Generate invalid emails
      (invalidEmail) => {
        const result = validateEmail(invalidEmail);
        expect(result.isValid).toBe(false);
        expect(result.error).toBeDefined();
      }
    ),
    { numRuns: 100 }
  );
});

// Property 6: Successful login stores token
// Feature: h2s-frontend, Property 6: Successful login stores token
test('any successful login response stores token in localStorage', () => {
  fc.assert(
    fc.property(
      fc.string({ minLength: 20 }), // Generate random JWT-like tokens
      fc.emailAddress(),
      (token, email) => {
        // Mock successful login response
        const response = { token };
        handleLoginSuccess(response, email);
        
        expect(localStorage.getItem('token')).toBe(token);
      }
    ),
    { numRuns: 100 }
  );
});

// Property 14: Authenticated requests include token header
// Feature: h2s-frontend, Property 14: Authenticated requests include token header
test('all API requests include token when authenticated', () => {
  fc.assert(
    fc.property(
      fc.string({ minLength: 20 }), // Random token
      fc.constantFrom('/assistant/ask', '/auth/login', '/auth/register'), // Random endpoint
      (token, endpoint) => {
        localStorage.setItem('token', token);
        const config = apiClient.interceptors.request.handlers[0](
          { url: endpoint, headers: {} }
        );
        
        expect(config.headers.Authorization).toBe(`Bearer ${token}`);
      }
    ),
    { numRuns: 100 }
  );
});

// Property 23: Messages display in chronological order
// Feature: h2s-frontend, Property 23: Messages display in chronological order
test('messages are always sorted by timestamp', () => {
  fc.assert(
    fc.property(
      fc.array(fc.record({
        id: fc.uuid(),
        text: fc.string(),
        sender: fc.constantFrom('user', 'ai'),
        timestamp: fc.date()
      }), { minLength: 2, maxLength: 20 }),
      (messages) => {
        const sorted = sortMessagesByTimestamp(messages);
        
        for (let i = 1; i < sorted.length; i++) {
          expect(sorted[i].timestamp.getTime())
            .toBeGreaterThanOrEqual(sorted[i-1].timestamp.getTime());
        }
      }
    ),
    { numRuns: 100 }
  );
});
```

### Testing Coverage Goals

- **Unit Test Coverage**: Minimum 80% code coverage
- **Property Test Coverage**: All 28 correctness properties implemented
- **Integration Tests**: Key user flows (registration → login → chat)
- **E2E Tests**: Optional for hackathon, recommended for production

### Test Organization

```
src/
  __tests__/
    unit/
      components/
        Login.test.tsx
        Register.test.tsx
        ChatInterface.test.tsx
        MessageList.test.tsx
        ProtectedRoute.test.tsx
      contexts/
        AuthContext.test.tsx
        ChatContext.test.tsx
      services/
        api.test.ts
        auth.test.ts
        assistant.test.ts
    properties/
      auth.properties.test.ts
      chat.properties.test.ts
      validation.properties.test.ts
      api.properties.test.ts
    integration/
      userFlows.test.tsx
```

### Mocking Strategy

- **API Calls**: Mock with MSW (Mock Service Worker) for realistic HTTP mocking
- **localStorage**: Mock with jest-localstorage-mock
- **React Router**: Mock navigation with MemoryRouter in tests
- **Contexts**: Provide test-specific context values for isolated component testing

### Continuous Testing

- Run unit tests on every file save during development
- Run full test suite (unit + property) before commits
- Property tests catch edge cases that manual testing might miss
- Both test types are complementary and necessary for comprehensive coverage
