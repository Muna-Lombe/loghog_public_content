# loghog_public_content
# How to Connect Your App to LogHog

This guide provides a step-by-step process for connecting your application to the LogHog monitoring system. Once connected, your application's logs will be streamed, stored, and available for analysis in the LogHog dashboard.

## Table of Contents

1.  [Prerequisites](#prerequisites)
2.  [Step 1: Create an App and Get Your Token](#step-1-create-an-app-and-get-your-token)
3.  [Step 2: Use Client Examples (SDK - Future Development)](#step-2-use-client-examples-sdk-future-development)
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

## Step 2: Use Client Examples (SDK - Future Development)

To integrate LogHog logging into your application, you can utilize the provided client *examples*. These examples demonstrate how to construct log payloads and communicate with the LogHog Worker API. A fully-fledged, officially supported SDK is planned for future development to further simplify integration.

### Where to Find the Client Examples

You can find various language-specific client examples in the `SETUP_GUIDE.md` file within the LogHog project, specifically under the section **"6. Client Libraries & SDKs"**.

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
  'https://loghog.your-domain.workers.dev',
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
    api_url='https://loghog.your-domain.workers.dev',
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

