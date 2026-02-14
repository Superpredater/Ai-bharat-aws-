# Design Document: Student Learning Enhancements

## Overview

This design extends the H2S Learning Assistant with three integrated capabilities: a Progress Dashboard for visualizing learning metrics, an Interactive Code Editor for hands-on practice, and an AI Code Review System for intelligent feedback. The design maintains the existing three-tier architecture (React frontend, Node.js backend, Python AI service) while adding new components, API endpoints, database tables, and UI pages.

The system will track student activities automatically, store code submissions and reviews, execute code safely in isolated environments, and provide contextual AI assistance across all features. All enhancements will support demo mode operation without requiring OpenAI API credentials.

## Architecture

### System Architecture

The enhanced system maintains the existing three-tier architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                     React Frontend (Vite)                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Progress   │  │     Code     │  │     Chat     │      │
│  │   Dashboard  │  │    Editor    │  │  Interface   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│         │                  │                  │              │
│         └──────────────────┴──────────────────┘              │
│                            │                                 │
└────────────────────────────┼─────────────────────────────────┘
                             │ HTTP/REST
┌────────────────────────────┼─────────────────────────────────┐
│                  Node.js Backend (Express)                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Progress   │  │     Code     │  │  Assistant   │      │
│  │     API      │  │  Execution   │  │     API      │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│         │                  │                  │              │
│         │                  │                  │              │
│    ┌────┴────────────────────────────────┐   │              │
│    │       SQLite Database               │   │              │
│    │  - users                            │   │              │
│    │  - progress_metrics                 │   │              │
│    │  - code_submissions                 │   │              │
│    │  - code_reviews                     │   │              │
│    │  - saved_snippets                   │   │              │
│    └─────────────────────────────────────┘   │              │
│                                               │              │
└───────────────────────────────────────────────┼──────────────┘
                                                │ HTTP
┌───────────────────────────────────────────────┼──────────────┐
│                Python AI Service (FastAPI)                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │     Chat     │  │     Code     │  │    Context   │      │
│  │    Agent     │  │    Review    │  │   Analysis   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└──────────────────────────────────────────────────────────────┘
```

### Component Interaction Flow

**Progress Dashboard Flow:**
1. Frontend requests progress data from Node backend
2. Backend queries SQLite for metrics within time range
3. Backend aggregates data by day/week/month
4. Frontend renders line graphs using charting library

**Code Execution Flow:**
1. Student writes code in Code_Editor component
2. Frontend sends code + language to Node backend
3. Backend spawns isolated process for execution
4. Backend captures output/errors with timeout
5. Frontend displays results in output panel

**Code Review Flow:**
1. Student submits code for review
2. Node backend stores submission in database
3. Backend forwards code to Python AI service
4. AI service analyzes syntax and logic
5. AI service returns structured feedback
6. Backend stores review in database
7. Frontend displays feedback in sections

## Components and Interfaces

### Frontend Components

#### ProgressDashboard Component

**Purpose:** Display student learning metrics as interactive line graphs

**Props:**
- `userId: string` - Current authenticated user ID

**State:**
- `metrics: ProgressMetrics` - Loaded progress data
- `timeRange: 'daily' | 'weekly' | 'monthly'` - Selected time filter
- `loading: boolean` - Data loading state
- `error: string | null` - Error message

**Key Methods:**
- `fetchProgressData(timeRange)` - Load metrics from API
- `handleTimeRangeChange(range)` - Update time filter
- `renderLineGraph(metricType, data)` - Render chart for metric

**API Calls:**
- `GET /api/progress?userId={id}&range={range}` - Fetch progress metrics

**UI Elements:**
- Time range selector (Daily/Weekly/Monthly buttons)
- Four line graphs (questions, submissions, topics, accuracy)
- Empty state message when no data exists
- Hover tooltips showing exact values

#### CodeEditor Component

**Purpose:** Interactive code editor with syntax highlighting and execution

**Props:**
- `userId: string` - Current authenticated user ID
- `initialCode: string` - Pre-loaded code content
- `initialLanguage: string` - Pre-selected language

**State:**
- `code: string` - Current editor content
- `language: 'python' | 'javascript' | 'java' | 'cpp'` - Selected language
- `output: ExecutionResult | null` - Execution results
- `executing: boolean` - Execution in progress
- `savedSnippets: CodeSnippet[]` - User's saved code

**Key Methods:**
- `handleCodeChange(newCode)` - Update editor content
- `handleLanguageChange(lang)` - Switch programming language
- `executeCode()` - Send code for execution
- `saveSnippet(name)` - Persist code to database
- `loadSnippet(snippetId)` - Load saved code
- `requestCodeReview()` - Submit code for AI review

**API Calls:**
- `POST /api/code/execute` - Execute code
- `POST /api/code/review` - Request AI review
- `POST /api/code/save` - Save code snippet
- `GET /api/code/snippets?userId={id}` - Load saved snippets

**UI Elements:**
- Language selector dropdown
- Code editor with syntax highlighting (using Monaco Editor or CodeMirror)
- Run button and Save button
- Output panel (stdout, stderr, execution time)
- Saved snippets sidebar
- Code review feedback panel

#### CodeReviewPanel Component

**Purpose:** Display structured AI feedback on code submissions

**Props:**
- `review: CodeReview` - Review data from API

**State:**
- `expandedSections: Set<string>` - Which sections are expanded

**Key Methods:**
- `toggleSection(sectionName)` - Expand/collapse section

**UI Elements:**
- Syntax errors section with line numbers
- Logic errors section with explanations
- Suggestions section with best practices
- Positive feedback when code is correct

#### EnhancedChatPanel Component

**Purpose:** Contextual AI assistance integrated with code editor

**Props:**
- `userId: string` - Current user ID
- `codeContext: string | null` - Current code in editor
- `mode: 'general' | 'code-help'` - Chat mode

**State:**
- `messages: Message[]` - Chat history
- `inputText: string` - Current input
- `loading: boolean` - Waiting for response

**Key Methods:**
- `sendMessage(text, includeCodeContext)` - Send message to AI
- `handleContextualHelp()` - Request help with current code

**API Calls:**
- `POST /api/assistant/ask` - Send message (existing endpoint, enhanced)

### Backend API Endpoints

#### Progress API

**GET /api/progress**

Query Parameters:
- `userId: string` (required)
- `range: 'daily' | 'weekly' | 'monthly'` (required)

Response:
```json
{
  "questionsAsked": [
    {"date": "2024-01-15", "count": 12},
    {"date": "2024-01-16", "count": 8}
  ],
  "codeSubmissions": [
    {"date": "2024-01-15", "count": 5},
    {"date": "2024-01-16", "count": 3}
  ],
  "topicsCovered": [
    {"date": "2024-01-15", "topics": ["arrays", "loops"]},
    {"date": "2024-01-16", "topics": ["functions"]}
  ],
  "accuracyScore": [
    {"date": "2024-01-15", "score": 0.75},
    {"date": "2024-01-16", "score": 0.80}
  ]
}
```

**POST /api/progress/track**

Request Body:
```json
{
  "userId": "string",
  "metricType": "questions_asked | code_submissions | topic_covered",
  "metricValue": "string | number",
  "timestamp": "ISO8601 datetime"
}
```

Response:
```json
{
  "success": true,
  "metricId": "number"
}
```

#### Code Execution API

**POST /api/code/execute**

Request Body:
```json
{
  "userId": "string",
  "code": "string",
  "language": "python | javascript | java | cpp"
}
```

Response:
```json
{
  "success": true,
  "stdout": "string",
  "stderr": "string",
  "exitCode": "number",
  "executionTime": "number (milliseconds)",
  "submissionId": "number"
}
```

Error Response:
```json
{
  "success": false,
  "error": "timeout | security_violation | compilation_error",
  "message": "string"
}
```

#### Code Review API

**POST /api/code/review**

Request Body:
```json
{
  "submissionId": "number",
  "code": "string",
  "language": "string"
}
```

Response:
```json
{
  "reviewId": "number",
  "syntaxErrors": [
    {
      "line": "number",
      "message": "string",
      "severity": "error | warning"
    }
  ],
  "logicErrors": [
    {
      "description": "string",
      "suggestion": "string"
    }
  ],
  "suggestions": [
    {
      "category": "readability | performance | best_practices",
      "message": "string"
    }
  ],
  "overallFeedback": "string"
}
```

#### Code Snippets API

**POST /api/code/save**

Request Body:
```json
{
  "userId": "string",
  "snippetName": "string",
  "code": "string",
  "language": "string"
}
```

Response:
```json
{
  "success": true,
  "snippetId": "number"
}
```

**GET /api/code/snippets**

Query Parameters:
- `userId: string` (required)

Response:
```json
{
  "snippets": [
    {
      "id": "number",
      "name": "string",
      "language": "string",
      "code": "string",
      "createdAt": "ISO8601 datetime"
    }
  ]
}
```

**DELETE /api/code/snippets/:id**

Response:
```json
{
  "success": true
}
```

### Python AI Service Endpoints

#### Code Review Endpoint

**POST /ai/review**

Request Body:
```json
{
  "code": "string",
  "language": "string",
  "demoMode": "boolean"
}
```

Response:
```json
{
  "syntaxErrors": [...],
  "logicErrors": [...],
  "suggestions": [...],
  "overallFeedback": "string"
}
```

**Implementation Strategy:**
- In demo mode: Use AST parsing and pattern matching
- With OpenAI: Use LLM for deeper analysis
- Maintain rule-based fallbacks for common errors

#### Enhanced Chat Endpoint

**POST /ai/ask** (Enhanced existing endpoint)

Request Body:
```json
{
  "message": "string",
  "history": [...],
  "codeContext": "string | null",
  "mode": "general | code-help"
}
```

Response:
```json
{
  "response": "string"
}
```

## Data Models

### Database Schema

#### progress_metrics Table

```sql
CREATE TABLE progress_metrics (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    metric_type TEXT NOT NULL,  -- 'questions_asked', 'code_submissions', 'topic_covered', 'accuracy_score'
    metric_value TEXT NOT NULL,  -- JSON string for complex values
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_progress_user_time ON progress_metrics(user_id, timestamp);
CREATE INDEX idx_progress_type ON progress_metrics(metric_type);
```

#### code_submissions Table

```sql
CREATE TABLE code_submissions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    code_content TEXT NOT NULL,
    language TEXT NOT NULL,
    execution_result TEXT,  -- JSON string with stdout, stderr, exitCode
    success BOOLEAN DEFAULT 0,
    execution_time INTEGER,  -- milliseconds
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_submissions_user_time ON code_submissions(user_id, timestamp);
```

#### code_reviews Table

```sql
CREATE TABLE code_reviews (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    submission_id INTEGER NOT NULL,
    syntax_feedback TEXT,  -- JSON array of syntax errors
    logic_feedback TEXT,   -- JSON array of logic errors
    suggestions TEXT,      -- JSON array of suggestions
    overall_feedback TEXT,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (submission_id) REFERENCES code_submissions(id) ON DELETE CASCADE
);

CREATE INDEX idx_reviews_submission ON code_reviews(submission_id);
```

#### saved_snippets Table

```sql
CREATE TABLE saved_snippets (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    snippet_name TEXT NOT NULL,
    code_content TEXT NOT NULL,
    language TEXT NOT NULL,
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_snippets_user ON saved_snippets(user_id);
```

### TypeScript Interfaces (Frontend)

```typescript
interface ProgressMetrics {
  questionsAsked: DataPoint[];
  codeSubmissions: DataPoint[];
  topicsCovered: TopicDataPoint[];
  accuracyScore: DataPoint[];
}

interface DataPoint {
  date: string;
  count: number;
}

interface TopicDataPoint {
  date: string;
  topics: string[];
}

interface ExecutionResult {
  success: boolean;
  stdout: string;
  stderr: string;
  exitCode: number;
  executionTime: number;
  submissionId: number;
}

interface CodeReview {
  reviewId: number;
  syntaxErrors: SyntaxError[];
  logicErrors: LogicError[];
  suggestions: Suggestion[];
  overallFeedback: string;
}

interface SyntaxError {
  line: number;
  message: string;
  severity: 'error' | 'warning';
}

interface LogicError {
  description: string;
  suggestion: string;
}

interface Suggestion {
  category: 'readability' | 'performance' | 'best_practices';
  message: string;
}

interface CodeSnippet {
  id: number;
  name: string;
  language: string;
  code: string;
  createdAt: string;
}
```

## Code Execution Security

### Isolation Strategy

Code execution must be isolated to prevent security vulnerabilities:

1. **Process Isolation:** Each execution runs in a separate child process
2. **Timeout Enforcement:** 10-second maximum execution time
3. **Resource Limits:** Memory and CPU limits per process
4. **Filesystem Restrictions:** No file system access except temp directories
5. **Network Restrictions:** No network access during execution

### Implementation Approach

**Node.js Backend:**
```javascript
const { spawn } = require('child_process');
const path = require('path');
const fs = require('fs').promises;
const crypto = require('crypto');

async function executeCode(code, language) {
  const executionId = crypto.randomBytes(16).toString('hex');
  const tempDir = path.join(__dirname, 'temp', executionId);
  
  await fs.mkdir(tempDir, { recursive: true });
  
  const config = getLanguageConfig(language);
  const filePath = path.join(tempDir, config.filename);
  
  await fs.writeFile(filePath, code);
  
  return new Promise((resolve, reject) => {
    const process = spawn(config.command, config.args(filePath), {
      timeout: 10000,
      cwd: tempDir,
      env: { ...process.env, NO_NETWORK: '1' }
    });
    
    let stdout = '';
    let stderr = '';
    
    process.stdout.on('data', (data) => { stdout += data; });
    process.stderr.on('data', (data) => { stderr += data; });
    
    process.on('close', async (code) => {
      await fs.rm(tempDir, { recursive: true, force: true });
      resolve({ stdout, stderr, exitCode: code });
    });
    
    process.on('error', async (err) => {
      await fs.rm(tempDir, { recursive: true, force: true });
      reject(err);
    });
  });
}

function getLanguageConfig(language) {
  const configs = {
    python: {
      filename: 'code.py',
      command: 'python3',
      args: (file) => [file]
    },
    javascript: {
      filename: 'code.js',
      command: 'node',
      args: (file) => [file]
    },
    java: {
      filename: 'Main.java',
      command: 'java',
      args: (file) => [file]
    },
    cpp: {
      filename: 'code.cpp',
      command: 'g++',
      args: (file) => [file, '-o', 'code.out', '&&', './code.out']
    }
  };
  return configs[language];
}
```

## Demo Mode AI Implementation

### Code Review in Demo Mode

The AI service will use rule-based analysis when OpenAI is unavailable:

**Syntax Error Detection:**
- Python: Use `ast.parse()` to detect syntax errors
- JavaScript: Use `esprima` or similar parser
- Java: Attempt compilation and capture errors
- C++: Attempt compilation and capture errors

**Logic Error Detection (Pattern Matching):**
- Infinite loops (while True without break)
- Uninitialized variables
- Array index out of bounds patterns
- Null/undefined access patterns
- Common algorithm mistakes (off-by-one errors)

**Suggestion Generation:**
- Check for naming conventions
- Detect missing error handling
- Identify code duplication
- Flag magic numbers
- Suggest const/final for unchanging values

**Python Implementation:**
```python
import ast
import re

def analyze_code_demo(code, language):
    if language == 'python':
        return analyze_python_demo(code)
    # Similar for other languages
    
def analyze_python_demo(code):
    syntax_errors = []
    logic_errors = []
    suggestions = []
    
    # Syntax check
    try:
        tree = ast.parse(code)
    except SyntaxError as e:
        syntax_errors.append({
            'line': e.lineno,
            'message': e.msg,
            'severity': 'error'
        })
        return {
            'syntaxErrors': syntax_errors,
            'logicErrors': [],
            'suggestions': [],
            'overallFeedback': 'Fix syntax errors before proceeding.'
        }
    
    # Logic checks
    for node in ast.walk(tree):
        # Check for infinite loops
        if isinstance(node, ast.While):
            if isinstance(node.test, ast.Constant) and node.test.value == True:
                has_break = any(isinstance(n, ast.Break) for n in ast.walk(node))
                if not has_break:
                    logic_errors.append({
                        'description': 'Potential infinite loop detected',
                        'suggestion': 'Add a break condition or change loop condition'
                    })
        
        # Check for bare except
        if isinstance(node, ast.ExceptHandler) and node.type is None:
            suggestions.append({
                'category': 'best_practices',
                'message': 'Avoid bare except clauses. Catch specific exceptions.'
            })
    
    # Naming conventions
    for node in ast.walk(tree):
        if isinstance(node, ast.FunctionDef):
            if not re.match(r'^[a-z_][a-z0-9_]*$', node.name):
                suggestions.append({
                    'category': 'readability',
                    'message': f'Function "{node.name}" should use snake_case naming'
                })
    
    overall = generate_overall_feedback(syntax_errors, logic_errors, suggestions)
    
    return {
        'syntaxErrors': syntax_errors,
        'logicErrors': logic_errors,
        'suggestions': suggestions,
        'overallFeedback': overall
    }

def generate_overall_feedback(syntax_errors, logic_errors, suggestions):
    if syntax_errors:
        return "Your code has syntax errors. Fix these first before running."
    elif logic_errors:
        return "Your code is syntactically correct but may have logic issues. Review the feedback carefully."
    elif suggestions:
        return "Great job! Your code works, but here are some suggestions to improve it."
    else:
        return "Excellent work! Your code looks good. Keep practicing!"
```



## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Property 1: Progress data filtering

*For any* time range selection (daily, weekly, monthly) and any set of progress metrics with timestamps, all displayed data points should fall within the selected time range boundaries.

**Validates: Requirements 1.4**

### Property 2: Metric persistence round-trip

*For any* progress metric (questions_asked, code_submissions, topic_covered, accuracy_score), after recording it to the database, querying for that user's metrics should return the recorded metric with a timestamp.

**Validates: Requirements 1.7, 6.7**

### Property 3: Code execution timeout enforcement

*For any* code that runs longer than 10 seconds, the execution engine should terminate the process and return a timeout error before 11 seconds elapse.

**Validates: Requirements 5.5**

### Property 4: Execution result structure completeness

*For any* code execution (successful or failed), the response should contain stdout, stderr, exitCode, and executionTime fields.

**Validates: Requirements 2.5, 5.9**

### Property 5: Code snippet round-trip preservation

*For any* valid code snippet with a name and language, saving it then loading it back should return code content identical to the original.

**Validates: Requirements 2.7, 2.8, 2.9**

### Property 6: Security isolation enforcement

*For any* code that attempts to access restricted resources (file system outside temp directory, network, system calls), the execution engine should block the operation and return a security error.

**Validates: Requirements 2.11, 5.6**

### Property 7: Code review response structure

*For any* code submission sent for review, the response should contain syntaxErrors, logicErrors, suggestions, and overallFeedback sections (arrays may be empty but fields must exist).

**Validates: Requirements 3.1, 3.2, 3.9**

### Property 8: Syntax error location reporting

*For any* code with known syntax errors, the review feedback should include line numbers indicating where each error occurs.

**Validates: Requirements 3.3**

### Property 9: Review persistence

*For any* code review generated by the AI service, querying the database with the submission ID should return the complete review data including all feedback sections.

**Validates: Requirements 3.10**

### Property 10: Code context inclusion

*For any* AI assistance request made while code is present in the editor, the API request to the AI service should include the current code content as context.

**Validates: Requirements 4.1, 4.7**

### Property 11: Conversation history continuity

*For any* authenticated user session, navigating between features (chat, code editor, dashboard) should preserve the conversation history without loss of messages.

**Validates: Requirements 4.8**

### Property 12: Memory limit enforcement

*For any* code execution, the process memory usage should not exceed the configured limit, and attempts to allocate beyond the limit should result in termination.

**Validates: Requirements 5.7**

### Property 13: Output capture completeness

*For any* code that writes to stdout or stderr, all output should be captured and returned in the execution result without truncation (up to reasonable size limits).

**Validates: Requirements 5.8**

### Property 14: Automatic metric incrementation

*For any* student activity (sending message, submitting code), the corresponding progress metric counter should increase by exactly 1.

**Validates: Requirements 6.1, 6.2**

### Property 15: Accuracy score calculation correctness

*For any* set of code submissions for a user, the calculated accuracy_score should equal the number of successful submissions divided by the total number of submissions.

**Validates: Requirements 6.5**

### Property 16: Metric aggregation correctness

*For any* set of progress metrics with timestamps, aggregating by day/week/month should group all metrics with timestamps in the same period together with no omissions or duplicates.

**Validates: Requirements 6.6**

### Property 17: SPA navigation without reload

*For any* navigation link click within the application, the browser should not perform a full page reload (no HTTP GET request for HTML).

**Validates: Requirements 7.2**

### Property 18: Authentication state persistence

*For any* authenticated user, navigating between any features should maintain the authentication token and user identity without requiring re-login.

**Validates: Requirements 7.3**

### Property 19: Active feature highlighting

*For any* currently displayed feature page, the navigation menu should visually indicate that feature as active.

**Validates: Requirements 7.4**

### Property 20: UI state persistence across navigation

*For any* UI state (open panels, selected tabs, editor content), navigating away from a feature and returning should restore the previous state.

**Validates: Requirements 7.7**

### Property 21: Foreign key referential integrity

*For any* attempt to insert a record with a foreign key reference to a non-existent parent record, the database should reject the operation with a constraint violation error.

**Validates: Requirements 8.6**

### Property 22: Demo mode contextual hints

*For any* code structure analyzed in demo mode, generated hints should reference elements actually present in the code (variable names, function names, control structures).

**Validates: Requirements 9.4**

### Property 23: Language-specific syntax highlighting

*For any* programming language selection, the editor should apply syntax highlighting rules appropriate to that language (keywords, strings, comments colored correctly).

**Validates: Requirements 10.5**

### Property 24: Character and line count accuracy

*For any* code content in the editor, the displayed character count should equal the actual string length, and the line count should equal the number of newline characters plus one.

**Validates: Requirements 10.6**

### Property 25: Error line highlighting accuracy

*For any* execution result containing error line numbers, the editor should highlight exactly those line numbers and no others.

**Validates: Requirements 10.9**

## Error Handling

### Frontend Error Handling

**Network Errors:**
- Display user-friendly messages when API calls fail
- Provide retry mechanisms for transient failures
- Show offline indicators when backend is unreachable

**Validation Errors:**
- Validate code snippet names before saving (non-empty, reasonable length)
- Validate language selection before execution
- Prevent submission of empty code for review

**State Errors:**
- Handle missing authentication gracefully (redirect to login)
- Handle corrupted local storage data (clear and reset)
- Handle race conditions in concurrent API calls

### Backend Error Handling

**Database Errors:**
- Catch and log all database operation failures
- Return appropriate HTTP status codes (500 for server errors)
- Implement transaction rollback for multi-step operations

**Execution Errors:**
- Catch process spawn failures (missing interpreters/compilers)
- Handle timeout errors gracefully
- Clean up temporary files even on error
- Prevent resource leaks from failed executions

**AI Service Errors:**
- Implement fallback to demo mode if AI service is down
- Retry failed AI requests with exponential backoff
- Cache recent AI responses to reduce load

### Python AI Service Error Handling

**Parsing Errors:**
- Catch syntax errors during AST parsing
- Return structured error information
- Provide helpful error messages for unsupported languages

**Analysis Errors:**
- Handle edge cases in pattern matching
- Gracefully degrade when analysis fails
- Always return valid response structure even on partial failure

## Testing Strategy

### Dual Testing Approach

This feature requires both unit testing and property-based testing for comprehensive coverage:

**Unit Tests:** Verify specific examples, edge cases, and error conditions
- Test empty progress data display (edge case from 1.6)
- Test each supported language execution (examples from 5.1-5.4)
- Test database table creation (examples from 8.1-8.5)
- Test demo mode activation (examples from 9.1-9.7)
- Test keyboard shortcuts (example from 10.2)
- Test navigation menu structure (example from 7.1)
- Test code review with no errors (example from 3.7)

**Property Tests:** Verify universal properties across all inputs
- All 25 correctness properties listed above
- Each property test should run minimum 100 iterations
- Use property-based testing library (fast-check for JavaScript/TypeScript, Hypothesis for Python)

### Property-Based Testing Configuration

**JavaScript/TypeScript (Frontend & Backend):**
- Library: fast-check
- Configuration: 100 iterations minimum per test
- Tag format: `// Feature: student-learning-enhancements, Property N: [property text]`

**Python (AI Service):**
- Library: Hypothesis
- Configuration: 100 examples minimum per test
- Tag format: `# Feature: student-learning-enhancements, Property N: [property text]`

### Test Organization

**Frontend Tests:**
```
h2s-frontend/src/
  components/
    __tests__/
      ProgressDashboard.test.jsx
      ProgressDashboard.property.test.jsx
      CodeEditor.test.jsx
      CodeEditor.property.test.jsx
      CodeReviewPanel.test.jsx
```

**Backend Tests:**
```
node-backend/
  tests/
    unit/
      progress.test.js
      codeExecution.test.js
      codeReview.test.js
    property/
      progress.property.test.js
      codeExecution.property.test.js
      security.property.test.js
```

**AI Service Tests:**
```
python-ai/
  tests/
    test_code_review.py
    test_code_review_properties.py
    test_demo_mode.py
```

### Integration Testing

**End-to-End Flows:**
1. Complete learning session: Login → Ask questions → Write code → Get review → View progress
2. Code snippet workflow: Write code → Save snippet → Load snippet → Verify content
3. Progress tracking: Perform activities → View dashboard → Verify metrics
4. Demo mode operation: Disable OpenAI → Submit code → Verify rule-based review

### Performance Testing

**Load Testing:**
- Concurrent code executions (10+ simultaneous)
- Large code submissions (10,000+ lines)
- High-frequency metric recording (100+ events/second)
- Dashboard rendering with large datasets (1000+ data points)

**Timeout Testing:**
- Verify 10-second execution timeout
- Verify 5-second API response time for code execution
- Test timeout cleanup (no zombie processes)

### Security Testing

**Isolation Testing:**
- Attempt file system access outside temp directory
- Attempt network connections during execution
- Attempt system command execution
- Verify all attempts are blocked

**Input Validation:**
- SQL injection attempts in code content
- XSS attempts in snippet names
- Path traversal attempts in file operations
- Oversized input handling

## Implementation Notes

### Technology Choices

**Code Editor Library:**
- Recommended: Monaco Editor (powers VS Code)
- Alternative: CodeMirror 6
- Features needed: Syntax highlighting, line numbers, keyboard shortcuts

**Charting Library:**
- Recommended: Recharts (React-friendly)
- Alternative: Chart.js with react-chartjs-2
- Features needed: Line charts, hover tooltips, responsive

**Code Execution:**
- Use Node.js child_process.spawn for isolation
- Implement resource limits using OS-level controls
- Clean up temp directories after each execution

**Demo Mode AI:**
- Python: Use ast module for parsing
- JavaScript: Use esprima for parsing
- Pattern matching: Regular expressions and AST traversal
- Knowledge base: JSON file with common patterns

### Deployment Considerations

**Database Migrations:**
- Create migration script for new tables
- Ensure backward compatibility with existing data
- Add indexes after table creation for performance

**Environment Variables:**
- `CODE_EXECUTION_TIMEOUT`: Configurable timeout (default 10000ms)
- `MAX_MEMORY_MB`: Memory limit per execution (default 512MB)
- `TEMP_DIR`: Directory for code execution (default ./temp)
- `OPENAI_API_KEY`: Optional for enhanced AI (empty = demo mode)

**Resource Management:**
- Implement cleanup cron job for old temp files
- Monitor disk space usage
- Implement rate limiting for code execution (prevent abuse)

### Future Enhancements

**Potential Extensions:**
- Collaborative coding (multiple students on same code)
- Code comparison (diff view for before/after)
- Achievement system (badges for milestones)
- Leaderboards (optional, privacy-respecting)
- Export progress reports (PDF/CSV)
- Custom test cases for code validation
- Video tutorials integrated with code editor
- Peer code review system
