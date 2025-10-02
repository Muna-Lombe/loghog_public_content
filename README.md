# LogHog | The Log Monitor For Everyone
## How to Connect Your App to LogHog

This guide provides a step-by-step process for connecting your application to the LogHog monitoring system. Once connected, your application's logs will be streamed, stored, and available for analysis in the LogHog dashboard.

## Table of Contents

1.  [Prerequisites](#prerequisites)
2.  [Step 1: Create an App and Get Your Token](#step-1-create-an-app-and-get-your-token)
3.  [Step 2: Custom LogHog Client Implementation Guide](#step-2-custom-loghog-client-implementation-guide)
4.  [Step 3: Instrument Your Application](#step-3-instrument-your-application)
5.  [Step 4: Verify the Connection](#step-4-verify-the-connection)
6.  [Advanced: Sending Structured Logs](#advanced-sending-structured-logs)

---

## Prerequisites

Before you begin, ensure you have the following:

*   A running instance of the LogHog Worker.
*   The API URL of your deployed LogHog Worker (e.g., `https://apihog.katundu.org/`).
*   Access to the Supabase database connected to your LogHog instance.

---

## Step 1: Create an App and Get Your Token

Every application that sends logs to LogHog must be registered and have a unique authentication token. You can easily create and manage your applications and their tokens directly from the LogHog dashboard.

### 1. Log in to the LogHog Dashboard

Navigate to your LogHog dashboard (e.g., `https://loghog.katundu.org/dashboard`) and log in with your user account.

### 2. Add a New Application

On the Dashboard page, click the "Add Application" button. Enter a descriptive name for your application and click "Create App". A new application will be listed on your dashboard.

### 3. Retrieve Your App Token

Click on your newly created application to go to its detail view. Navigate to the "Token Management" tab. Here, you will see your unique **App Token**. Copy this token securely, as it is essential for authenticating your application's logs with LogHog.

**Important**: Keep your **App Token** confidential. It is used by your application to send logs to LogHog.

---

## Step 2: Custom LogHog Client Implementation Guide

While a dedicated SDK is under future development, you can easily create your own `LogHogClient` using the following guidelines. This ensures your logs are formatted correctly and sent to the LogHog Worker API, allowing for effective monitoring and analysis.

### Recommended `LogHogClient` Structure

Your custom `LogHogClient` should, at a minimum, implement a constructor to initialize with the LogHog API URL and your application token, along with a `log` method to send structured log data. Consider adding convenience methods for different log levels to streamline usage.

```javascript
class LogHogClient {
  /**
   * Initializes the LogHogClient.
   * @param {string} apiUrl - The base URL of your LogHog Worker API (e.g., "https://apihog.katundu.org"). In a Create React App deployment, this value will be automatically sourced from `window.REACT_APP_WORKER_API` in production, or fallback to a local URL in development.
   * @param {string} token - Your application's unique authentication token, obtained from the LogHog dashboard.
   *                          This token should be kept confidential and ideally loaded from environment variables or a secure secret store.
   */
  constructor(apiUrl, token) {
    if (!apiUrl || !token) {
      throw new Error('LogHogClient requires both apiUrl and token.');
    }
    this.apiUrl = apiUrl.endsWith('/') ? apiUrl.slice(0, -1) : apiUrl; // Ensure no trailing slash
    this.token = token;
    this.logEndpoint = `${this.apiUrl}/api/logs`;
  }

  /**
   * Sends a structured log entry to the LogHog API.
   * @param {string} level - The log level (e.g., 'info', 'warn', 'error', 'debug', 'fatal'). This is a required field.
   * @param {string} message - A concise, human-readable summary of the log event. This is a required field.
   * @param {object} [data={}] - An optional object containing detailed structured data for the log. This object's contents will be deeply compressed and stored.
   *                               It can also contain special fields like `category`, `trace_id`, `span_id`, `template_id`, `params`, and `tags` which are extracted and indexed for querying.
   */
  async log(level, message, data = {}) {
    if (!level || !message) {
      console.error('LogHog: Log level and message are required.');
      return; // Or throw an error, depending on desired strictness
    }

    // Construct the payload according to LogHog's expected schema.
    // Fields like 'category', 'trace_id', 'span_id', 'template_id', 'params', 'tags'
    // are extracted from 'data' for indexing and filtering, and also included in 'body'.
    const payload = {
      timestamp: new Date().toISOString(),      // ISO 8601 format (e.g., "2024-01-01T12:00:00.000Z"). Crucial for chronological ordering.
      level: level,                             // Log severity: 'info', 'warn', 'error', 'debug', 'fatal'. Helps in filtering and severity analysis.
      message: message,                         // A brief, descriptive summary of the event. Essential for quick understanding.
      body: data,                               // The comprehensive JSON object containing all detailed, structured data related to the log event.
      category: data.category || 'general',       // Optional: A high-level classification of the log (e.g., 'auth', 'database', 'network'). Useful for broad filtering.
      trace_id: data.trace_id || undefined,     // Optional: A unique identifier to correlate multiple log entries belonging to a single request or transaction across services.
      span_id: data.span_id || undefined,       // Optional: An identifier for a specific operation within a trace, useful for distributed tracing contexts.
      template_id: data.template_id || undefined, // Optional: An identifier for a predefined log message template, useful for grouping similar logs.
      params: data.params || undefined,         // Optional: An object containing parameters that fill a predefined log message template.
      tags: data.tags || undefined              // Optional: An object of key-value pairs for additional metadata and filtering (e.g., { environment: 'production', service: 'payments-api' }).
    };

    try {
      const response = await fetch(this.logEndpoint, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${this.token}` // Essential for authenticating your application with LogHog.
        },
        body: JSON.stringify(payload)
      });

      if (!response.ok) {
        // Robust error handling: capture status and response body for debugging.
        const errorDetails = await response.text();
        const errorMessage = `LogHog API error: ${response.status} - ${response.statusText}. Details: ${errorDetails}`;
        console.error('Failed to send log to LogHog:', errorMessage);
        throw new Error(errorMessage); // Re-throw to allow the calling application to react to failed log submissions.
      }

      return await response.json(); // Returns the API response, typically a confirmation of log ingestion.
    } catch (error) {
      // Network errors or issues prior to receiving an HTTP response.
      console.error('LogHog: Network or unexpected error when sending log:', error);
      throw error; // Ensure critical errors are propagated.
    }
  }

  // Convenience methods (optional, but highly recommended for ergonomic logging)
  /**
   * Logs an informational message.
   * @param {string} message - The log message.
   * @param {object} [data={}] - Additional structured data.
   */
  info(message, data) { return this.log('info', message, data); }

  /**
   * Logs a warning message.
   * @param {string} message - The log message.
   * @param {object} [data={}] - Additional structured data.
   */
  warn(message, data) { return this.log('warn', message, data); }

  /**
   * Logs an error message.
   * @param {string} message - The log message.
   * @param {object} [data={}] - Additional structured data.
   */
  error(message, data) { return this.log('error', message, data); }

  /**
   * Logs a debug message.
   * @param {string} message - The log message.
   * @param {object} [data={}] - Additional structured data.
   */
  debug(message, data) { return this.log('debug', message, data); }

  /**
   * Logs a fatal error message (indicating a critical, unrecoverable issue).
   * @param {string} message - The log message.
   * @param {object} [data={}] - Additional structured data.
   */
  fatal(message, data) { return this.log('fatal', message, data); }
}
```

### Available Log Templates

LogHog provides several predefined log templates that you can use to standardize your log messages. When using a template, provide its `name` in the `template` field of your log payload, along with any required `params`.

| Template Name        | Description                       | Template String                               |
|----------------------|-----------------------------------|-----------------------------------------------|
| `HTTP_INFO`          | Informational HTTP request log    | `HTTP {{statusCode}} {{method}} {{path}}`     |
| `HTTP_ERROR`         | Error in HTTP request             | `HTTP Error {{statusCode}} {{method}} {{path}}` |
| `AUTH_LOGIN_SUCCESS` | User login successful             | `User {{email}} logged in successfully`       |
| `AUTH_LOGIN_FAILURE` | User login failed                 | `Failed login attempt for user {{email}}`     |
| `DB_QUERY_SLOW`      | Database query was slow           | `Slow DB query on table {{table}} took {{duration}}ms` |

### Advanced Considerations for Custom Clients

When implementing your custom `LogHogClient`, consider the following advanced patterns to enhance its robustness, efficiency, and integration capabilities:

1.  **Asynchronous and Non-Blocking Operations**: Ensure your `log` method (and any convenience methods) operates asynchronously. This prevents log submission from blocking the main execution thread of your application, which is crucial for performance, especially in high-throughput systems. The `async`/`await` pattern demonstrated in the `log` method is a good starting point.

2.  **Retry Mechanisms**: Implement a retry strategy for network requests to the LogHog API. Transient network issues are common, and retrying failed log submissions (with an exponential backoff) can significantly increase reliability. For example, you might retry 3-5 times with increasing delays.

    *   **Example (Conceptual Retry Logic)**:
        ```javascript
        async function retryFetch(url, options, retries = 3) {
          for (let i = 0; i < retries; i++) {
            try {
              const response = await fetch(url, options);
              if (response.ok) return response;
              // Only retry for certain status codes (e.g., 5xx, or network issues)
              if (response.status >= 500 || response.status === 429) { // 429 Too Many Requests
                await new Promise(res => setTimeout(res, Math.pow(2, i) * 1000)); // Exponential backoff
                continue;
              }
              throw new Error(`HTTP Error: ${response.status}`);
            } catch (error) {
              if (i === retries - 1) throw error; // Re-throw on last attempt
              // Log retry attempt
              await new Promise(res => setTimeout(res, Math.pow(2, i) * 1000));
            }
          }
        }
        // ... then use in your log method:
        // const response = await retryFetch(this.logEndpoint, {...});
        ```

3.  **Batching and Buffering Logs**: For applications generating a very high volume of logs, sending each log individually can be inefficient. Consider implementing a buffer within your client that collects logs over a short period (e.g., a few seconds) or until a certain number of logs is reached, and then sends them in a single batch request to a hypothetical `/api/logs/batch` endpoint. This would require a corresponding batch endpoint in the LogHog Worker.

4.  **Context Propagation (Trace & Span IDs)**: Leverage `trace_id` and `span_id` for distributed tracing. If your application uses a tracing system (like OpenTelemetry or OpenTracing), ensure these IDs are extracted from the current request context and passed to the `log` method's `data` object. This allows you to correlate logs across different services and functions involved in a single operation.

    *   **Best Practice**: Generate `trace_id` at the entry point of a request and pass it down through all subsequent operations. Generate `span_id` for individual operations within that trace.
    *   **Implementation Guidance for Trace & Span IDs**:
        To ensure comprehensive and correct setup of `trace_id` and `span_id`, consider the following principles and practices:

        **a. Understanding Trace Context:**
        `trace_id` and `span_id` are part of a broader concept called "trace context." This context encapsulates information about a single, end-to-end request or transaction as it flows through various services. Properly propagating this context allows observability tools like LogHog to reconstruct the full journey of a request.

        **b. Generating `trace_id` at the Entry Point:**
        The `trace_id` should be a globally unique identifier for an entire distributed trace. It is generated once at the very beginning of a request's lifecycle (e.g., when an API gateway receives an incoming request, or a message queue consumer starts processing a message). Subsequent services involved in the request should *reuse* this `trace_id`.

        *   **How to Generate:** Use a robust UUID (Universally Unique Identifier) generator. Most programming languages have built-in UUID functions (e.g., `crypto.randomUUID()` in Node.js, `uuid.uuid4()` in Python).

        *   **Example (Conceptual - Node.js HTTP Server Entry Point):**
            ```javascript
            const http = require('http');
            const { randomUUID } = require('crypto');

            http.createServer((req, res) => {
              const traceId = req.headers['x-trace-id'] || randomUUID(); // Use existing or generate new
              // Attach traceId to request context or pass it to initial logging calls
              req.traceId = traceId;
              // ... proceed with request handling ...
            }).listen(8080);
            ```

        **c. Generating `span_id` for Operations (Parent-Child Relationships):**
        A `span_id` represents a single logical unit of work within a trace (e.g., an API call, a database query, a function execution). Spans form a directed acyclic graph, where a parent span can have multiple child spans.

        *   **First Span (`root` span):** For each new `trace_id`, the very first operation should generate its own `span_id`. This is often referred to as the "root" span for that particular service's part of the trace.

        *   **Child Spans:** When an operation (parent span) initiates another operation (child span), the child span should generate its own unique `span_id` and also reference its parent's `span_id` (this is implicitly handled by passing the parent's context). LogHog's current schema primarily focuses on `trace_id` and the immediate `span_id`, but understanding the parent-child relationship is key for full tracing systems.

        *   **How to Generate:** Like `trace_id`, `span_id` should also be a unique identifier (often a shorter, simpler string or UUID). It must be unique within its trace.

        *   **Example (Conceptual - Node.js function within a traced request):**
            ```javascript
            async function getUserData(req, userId) {
              const traceId = req.traceId; // Inherit from incoming request
              const spanId = randomUUID().substring(0, 16); // Generate a unique span ID

              // Log with trace and span IDs
              loghog.info('Fetching user data', { trace_id: traceId, span_id: spanId, userId });

              // Simulate DB call or other operation
              await db.query('SELECT * FROM users WHERE id = $1', [userId]);
              loghog.debug('User data fetched', { trace_id: traceId, span_id: spanId, userId, dbOperation: 'read' });

              return { id: userId, name: 'John Doe' };
            }
            ```

        **d. Propagation Across Service Boundaries:**
        The critical step for distributed tracing is propagating the `trace_id` (and potentially `span_id` or a full trace context) when making calls to other services.

        *   **HTTP Headers (Recommended):** The most common method is to use standardized HTTP headers. The `W3C Trace Context` specification defines `traceparent` and `tracestate` headers, but simpler custom headers like `X-Trace-ID` are also widely used.

        *   **Example (Conceptual - Making an HTTP call to another service):**
            ```javascript
            async function callPaymentService(traceId, currentSpanId, amount) {
              const newSpanId = randomUUID().substring(0, 16); // New span for this outgoing call
              loghog.info('Calling payment service', { trace_id: traceId, span_id: newSpanId, parent_span_id: currentSpanId, amount });

              const response = await fetch('https://payments.example.com/process', {
                method: 'POST',
                headers: {
                  'Content-Type': 'application/json',
                  'X-Trace-ID': traceId,          // Propagate trace ID
                  'X-Parent-Span-ID': currentSpanId, // Propagate parent span ID (for context)
                  // Optionally, also send a new X-Span-ID for this specific outgoing call
                },
                body: JSON.stringify({ amount })
              });
              // ... handle response ...
            }
            ```

        *   **Message Queues:** For asynchronous communication, include `trace_id` and `span_id` in the message payload or as message headers.

        **e. Utilizing Tracing Libraries/APIs:**
        Manually managing `trace_id` and `span_id` can become complex. Dedicated distributed tracing libraries and APIs (e.g., OpenTelemetry, OpenTracing, or language-specific frameworks like Jaeger clients) can automate much of the trace context generation, propagation, and instrumentation. These libraries integrate with various frameworks and protocols to handle the boilerplate of tracing, allowing developers to focus on business logic.

        By following these guidelines, you can ensure that your application generates and propagates `trace_id` and `span_id` effectively, providing LogHog with the necessary context for powerful distributed tracing and observability.

5.  **Secure Configuration Management**: Emphasize that `apiUrl` and especially `token` should **never** be hardcoded in your application's source code. Instead, load them securely from:
    *   **Environment Variables**: For server-side applications and CI/CD pipelines.
    *   **Secret Management Services**: Such as AWS Secrets Manager, Google Secret Manager, Azure Key Vault, or HashiCorp Vault for production environments.
    *   **Build-time Configuration**: For client-side applications where environment variables are injected during the build process (as `REACT_APP_WORKER_API` is used in this project).

By incorporating these advanced considerations, your custom `LogHogClient` can become a more resilient, performant, and observable component of your application architecture.

### Client Examples for Various Languages

You can find concrete client *examples* in `SETUP_GUIDE.md` file within the LogHog project, specifically under the section **"6. Client Libraries & SDKs"**. These examples demonstrate the implementation of the `LogHogClient` structure in different programming languages.

Choose the client example that matches your application's language and copy it into your project:

*   **For Node.js / JavaScript**: Copy the `LogHogClient` class and save it as `loghog-client.js` in your project.
*   **For Python**: Copy the `LogHogClient` class and save it as `loghog_client.py`. You will also need the `requests` library: `pip install requests`
*   **For Go**: Copy the `Client` struct and its associated methods and save them as `loghog_client.go`.

---

## Step 3: Instrument Your Application

Configure your custom `LogHogClient` in your application with your LogHog API URL and the **App Token** you generated in Step 1. Ensure the `apiUrl` matches your deployed Cloudflare Worker's domain.

### Node.js Example

```javascript
const LogHogClient = require('./loghog-client.js');

// Configure the client
const loghog = new LogHogClient(
  'https://apihog.katundu.org',
  'paste-your-app-token-here'
);

// Send a simple log
loghog.info('User completed checkout process.');

// Send a log with a structured body
loghog.error('Payment processing failed.', {
  body: {
    userId: 'user-123',
    orderId: 'order-abc',
    paymentGateway: 'Stripe',
    errorCode: 'card_declined'
  },
  tags: {
    service: 'payments-api'
  }
});
```

### Python Example

```python
from loghog_client import LogHogClient

# Configure the client
loghog = LogHogClient(
    api_url='https://apihog.katundu.org',
    token='paste-your-app-token-here'
)

# Send a simple log
loghog.info('User completed checkout process.')

# Send a log with a structured body
loghog.error('Payment processing failed.', {
    'body': {
        'userId': 'user-123',
        'orderId': 'order-abc',
        'paymentGateway': 'Stripe',
        'errorCode': 'card_declined'
    },
    'tags': {
        'service': 'payments-api'
    }
})
```

---

## Step 4: Verify the Connection

1.  **Run Your Application**: Start your application that is now instrumented with the LogHog client.
2.  **Trigger a Log**: Perform an action in your app that sends a log (e.g., visit a webpage, process a request).
3.  **Check the Dashboard**: Open your LogHog dashboard. You should see the new logs appearing in real-time.
4.  **Inspect the Log**: Click the "View" button on a log entry to see the full, uncompressed, and formatted payload to verify that all data is being sent correctly.

---

## Advanced: Sending Structured Logs

LogHog is most powerful when you send structured data. The log payload can include several useful fields:

| Field         | Type    | Description                                           |
|---------------|---------|-------------------------------------------------------|
| `level`       | String  | `info`, `warn`, `error`, `debug`, `fatal`. Required.   |
| `message`     | String  | A short, human-readable summary of the log.           |
| `body`        | Object  | The full, detailed payload. This is what gets compressed. |
| `tags`        | Object  | Key-value pairs for filtering (e.g., `env:prod`).       |
| `trace_id`    | String  | An ID to correlate logs across a single request/trace. |
| `span_id`     | String  | An ID for a specific operation within a trace. |
| `category`    | String  | A high-level category for the log (e.g., `auth`).     |
| `template`    | Object  | `{ name: string, params: object }`. Name of a predefined log template and its parameters. |

### Example of a Rich Log Payload:

```javascript
loghog.error('Failed to process user image', {
  body: {
    userId: 'user-456',
    imageSize: 5242880, // in bytes
    imageType: 'image/jpeg',
    error: 'Image resolution too high.',
    maxResolution: '4096x4096'
  },
  tags: {
    service: 'image-processor',
    region: 'us-east-1',
    environment: 'production'
  },
  trace_id: 'trace-xyz-789',
  span_id: 'span-abc-456',
  category: 'media',
  template: { id: 'IMAGE_PROC_ERROR', params: { errorCode: 'resolution_too_high' } }
});
```

By providing rich, structured data, you can leverage LogHog's filtering, searching, and smart formatting capabilities to their fullest potential.
