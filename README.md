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
*   The API URL of your deployed LogHog Worker (e.g., `https://loghog.katundu.org/`).
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
   * @param {string} apiUrl - The base URL of your LogHog Worker API (e.g., "https://apihog.katundu.org").
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

### Client Examples for Various Languages

You can find concrete client *examples* in `SETUP_GUIDE.md` file within the LogHog project, specifically under the section **"6. Client Libraries & SDKs"**. These examples demonstrate the implementation of the `LogHogClient` structure in different programming languages.

Choose the client example that matches your application's language and copy it into your project:

*   **For Node.js / JavaScript**: Copy the `LogHogClient` class and save it as `loghog-client.js` in your project.
*   **For Python**: Copy the `LogHogClient` class and save it as `loghog_client.py`. You will also need the `requests` library: `pip install requests`
*   **For Go**: Copy the `Client` struct and its associated methods and save them as `loghog_client.go`.

---

## Step 3: Instrument Your Application

Configure the SDK in your application with your LogHog API URL and the **App Token** you generated in Step 1.

### Node.js Example

```javascript
const LogHogClient = require('./loghog-client.js');

// Configure the client
const loghog = new LogHogClient(
  'ttps://apihog.katundu.org',
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
| `category`    | String  | A high-level category for the log (e.g., `auth`).     |

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
  category: 'media'
});
```

By providing rich, structured data, you can leverage LogHog's filtering, searching, and smart formatting capabilities to their fullest potential.
