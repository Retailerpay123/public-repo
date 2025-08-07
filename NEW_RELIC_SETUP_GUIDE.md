# New Relic Setup Guide for Next.js Projects

## Overview
This guide provides step-by-step instructions to set up complete New Relic monitoring (server-side and browser-side) in a Next.js application.

## Prerequisites
- Next.js 13+ project (App Router)
- New Relic account with license key
- Node.js 18+

## Installation Steps

### 1. Install New Relic Package

```bash
npm install newrelic
```

### 2. Create New Relic Configuration File

Create `newrelic.js` in your project root:

```javascript
'use strict'

exports.config = {
  app_name: [process.env.NEW_RELIC_APP_NAME || 'Your-App-Name'],
  license_key: process.env.NEW_RELIC_LICENSE_KEY,
  logging: {
    level: process.env.NODE_ENV === 'development' ? 'info' : 'warn',
    enabled: true
  },
  distributed_tracing: {
    enabled: true
  },
  browser_monitoring: {
    enable: true,
    debug: process.env.NODE_ENV === 'development',
    loader: 'spa',  // Single Page Application mode for React/Next.js
    attributes: {
      enabled: true
    }
  },
  application_logging: {
    forwarding: {
      enabled: true
    },
    local_decorating: {
      enabled: true
    }
  },
  allow_all_headers: true,
  capture_params: true,
  attributes: {
    enabled: true,
    exclude: [
      'request.headers.cookie',
      'request.headers.authorization',
      'request.headers.proxyAuthorization'
    ]
  },
  error_collector: {
    enabled: true,
    capture_events: true
  },
  transaction_tracer: {
    enabled: true,
    transaction_threshold: 'apdex_f',
    record_sql: 'obfuscated'
  }
}
```

### 3. Update Next.js Configuration

Update `next.config.js`:

```javascript
// Import New Relic FIRST - before any other modules
require('./newrelic.js');

/** @type {import('next').NextConfig} */
const nextConfig = {
  // New Relic instrumentation
  webpack: (config, { isServer }) => {
    // Ensure New Relic is loaded first on server side
    if (isServer) {
      config.externals = config.externals || [];
      config.externals.push('newrelic');
    }
    return config;
  },
  // Enable server components (updated for Next.js 15)
  serverExternalPackages: ['newrelic']
}

module.exports = nextConfig
```

### 4. Update Package.json Scripts

Update the start script in `package.json`:

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "node -r ./newrelic.js ./node_modules/.bin/next start",
    "lint": "next lint"
  }
}
```

### 5. Create Browser Monitoring Component

Create `src/components/NewRelicBrowserAgent.tsx`:

```tsx
'use client';

import { useEffect, useState } from 'react';
import Script from 'next/script';

export default function NewRelicBrowserAgent() {
  const [browserScript, setBrowserScript] = useState<string>('');
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchBrowserScript = async () => {
      try {
        const response = await fetch('/api/newrelic-browser');
        const data = await response.json();
        
        if (data.success && data.browserScript) {
          setBrowserScript(data.browserScript);
        } else {
          console.warn('New Relic browser script not available:', data.error || 'Unknown error');
        }
      } catch (error) {
        console.error('Failed to fetch New Relic browser script:', error);
        // Fail silently in production to avoid breaking the page
      } finally {
        setLoading(false);
      }
    };

    fetchBrowserScript();
  }, []);

  if (loading || !browserScript) {
    return null;
  }

  return (
    <Script
      id="newrelic-browser-agent"
      strategy="afterInteractive"
      dangerouslySetInnerHTML={{
        __html: browserScript
      }}
    />
  );
}
```

### 6. Create API Route for Browser Script

Create `src/app/api/newrelic-browser/route.ts`:

```typescript
import { NextRequest, NextResponse } from 'next/server';

export async function GET(request: NextRequest) {
  try {
    // Import New Relic agent
    const newrelic = require('newrelic');
    
    // Get the browser monitoring script
    const browserScript = newrelic.getBrowserTimingHeader();
    
    if (!browserScript) {
      return NextResponse.json(
        { 
          success: false, 
          error: 'Browser script not available. Check New Relic configuration.' 
        },
        { status: 500 }
      );
    }

    return NextResponse.json({
      success: true,
      browserScript: browserScript
    });
    
  } catch (error) {
    console.error('Error generating New Relic browser script:', error);
    
    return NextResponse.json(
      { 
        success: false, 
        error: 'Failed to generate browser monitoring script' 
      },
      { status: 500 }
    );
  }
}
```

### 7. Update Root Layout

Update `src/app/layout.tsx` to include the browser agent:

```tsx
import React from 'react';
import NewRelicBrowserAgent from "../components/NewRelicBrowserAgent";

export const metadata = {
  title: "Your App Title",
  description: "Your app description",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <head>
        <NewRelicBrowserAgent />
      </head>
      <body>
        {children}
      </body>
    </html>
  );
}
```

## Environment Variables

### Required Environment Variables

Set these in your deployment environment:

```bash
# Required
NEW_RELIC_LICENSE_KEY=your_license_key_here

# Optional (defaults to app name in newrelic.js)
NEW_RELIC_APP_NAME=Your-Custom-App-Name
```

### Local Development

Create `.env.local` for local development:

```bash
NEW_RELIC_LICENSE_KEY=your_license_key_here
NEW_RELIC_APP_NAME=Your-App-Name-Dev
```

Add `.env.local` to your `.gitignore`:

```gitignore
# Local env files
.env.local
.env.*.local
```

## Deployment Configuration

### Vercel
Add environment variables in Vercel dashboard:
- Go to Project Settings → Environment Variables
- Add `NEW_RELIC_LICENSE_KEY` and `NEW_RELIC_APP_NAME`

### Docker
Add to your Dockerfile:

```dockerfile
ENV NEW_RELIC_LICENSE_KEY=your_license_key
ENV NEW_RELIC_APP_NAME=your_app_name
```

### Other Platforms
Set environment variables according to your platform's documentation.

## Verification Steps

### 1. Server-side Monitoring
1. Start your application
2. Make some requests to your API routes
3. Check New Relic APM dashboard for server metrics

### 2. Browser Monitoring
1. Open your application in browser
2. Open developer tools → Network tab
3. Look for New Relic browser agent script injection
4. Check New Relic Browser dashboard for client-side metrics

### 3. Debug Issues

**Common Issues:**

1. **No data in New Relic dashboard:**
   - Verify `NEW_RELIC_LICENSE_KEY` is set correctly
   - Check application logs for New Relic errors
   - Ensure `newrelic.js` is loaded before other modules

2. **Browser monitoring not working:**
   - Check if API route `/api/newrelic-browser` returns data
   - Verify browser agent component is included in layout
   - Check browser console for JavaScript errors

3. **Build errors:**
   - Ensure `newrelic` is in `serverExternalPackages` in `next.config.js`
   - Check that webpack configuration is correct

## Advanced Configuration

### Custom Attributes
Add custom attributes to transactions:

```javascript
// In your API routes or server components
const newrelic = require('newrelic');

newrelic.addCustomAttribute('userId', userId);
newrelic.addCustomAttribute('userType', userType);
```

### Custom Events
Track custom events:

```javascript
newrelic.recordCustomEvent('UserAction', {
  action: 'button_click',
  page: '/dashboard',
  userId: userId
});
```

### Error Tracking
Manually report errors:

```javascript
newrelic.noticeError(error, {
  userId: userId,
  action: 'api_call'
});
```

## Security Considerations

1. **Environment Variables**: Never commit license keys to version control
2. **CSP Headers**: If using Content Security Policy, whitelist New Relic domains:
   ```
   script-src 'self' *.newrelic.com js-agent.newrelic.com bam.nr-data.net;
   connect-src 'self' *.newrelic.com bam.nr-data.net;
   ```

3. **Data Privacy**: Review what data is being sent to New Relic and exclude sensitive information in the configuration

## Monitoring Best Practices

1. **Naming Conventions**: Use consistent app naming across environments (e.g., `MyApp-Production`, `MyApp-Staging`)
2. **Alerts**: Set up alerts for key metrics (response time, error rate, throughput)
3. **Dashboards**: Create custom dashboards for business-specific metrics
4. **SLA Monitoring**: Configure Apdex scores based on your performance requirements

## Troubleshooting

### Log Analysis
Enable detailed logging in development:

```javascript
// In newrelic.js
logging: {
  level: 'trace', // Use 'trace' for maximum detail
  enabled: true
}
```

### Health Check
Create a health check endpoint to verify New Relic integration:

```typescript
// src/app/api/health/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  try {
    const newrelic = require('newrelic');
    const agent = newrelic.agent;
    
    return NextResponse.json({
      newrelic: {
        connected: agent.config.agent_enabled,
        appName: agent.config.app_name,
        version: agent.version
      }
    });
  } catch (error) {
    return NextResponse.json({ error: 'New Relic not available' }, { status: 500 });
  }
}
```

This documentation provides everything needed to replicate New Relic monitoring setup in any new Next.js project.
