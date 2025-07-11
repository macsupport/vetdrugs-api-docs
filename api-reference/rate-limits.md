# Rate Limits

Understanding and managing VetDrugs API rate limits for optimal performance.

## Overview

The VetDrugs API implements rate limiting to ensure fair usage and maintain service quality for all users. Rate limits are applied per API key and are designed to accommodate typical veterinary practice workflows.

## Rate Limit Structure

### Current Limits

| Endpoint Category | Requests per Minute | Requests per Hour | Burst Limit |
|------------------|-------------------|------------------|-------------|
| **Drug Calculations** (`/api/calculate`) | 100 | 2,000 | 10 |
| **Drug Search** (`/api/drugs`) | 200 | 5,000 | 20 |
| **Drug Information** (`/api/drug-info/*`) | 150 | 3,000 | 15 |
| **Health Check** (`/api/health`) | 60 | 1,000 | 5 |
| **All Endpoints Combined** | 300 | 8,000 | 30 |

### Rate Limit Windows

- **Per Minute**: Rolling 60-second window
- **Per Hour**: Rolling 3600-second window  
- **Burst**: Maximum requests in a 10-second window

## Rate Limit Headers

Every API response includes rate limit information in the headers:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1625097600
X-RateLimit-Window: 60
X-RateLimit-Burst-Limit: 10
X-RateLimit-Burst-Remaining: 8
```

### Header Descriptions

| Header | Description | Example |
|--------|-------------|---------|
| `X-RateLimit-Limit` | Maximum requests allowed in the current window | `100` |
| `X-RateLimit-Remaining` | Requests remaining in the current window | `87` |
| `X-RateLimit-Reset` | Unix timestamp when the rate limit resets | `1625097600` |
| `X-RateLimit-Window` | Rate limit window in seconds | `60` |
| `X-RateLimit-Burst-Limit` | Maximum burst requests allowed | `10` |
| `X-RateLimit-Burst-Remaining` | Burst requests remaining | `8` |

## Rate Limit Exceeded Response

When rate limits are exceeded, the API returns a `429 Too Many Requests` status:

```json
{
  "error": "rate_limit_exceeded",
  "message": "Rate limit of 100 requests per minute exceeded",
  "limit": 100,
  "remaining": 0,
  "resetTime": "2025-07-11T19:00:00Z",
  "retryAfter": 45
}
```

### Response Fields

| Field | Description |
|-------|-------------|
| `error` | Error code: `rate_limit_exceeded` |
| `message` | Human-readable error message |
| `limit` | The rate limit that was exceeded |
| `remaining` | Always `0` when rate limited |
| `resetTime` | ISO 8601 timestamp when limit resets |
| `retryAfter` | Seconds to wait before retrying |

## Handling Rate Limits

### 1. Check Headers Before Making Requests

```javascript
class RateLimitAwareClient {
  constructor(apiKey) {
    this.apiKey = apiKey;
    this.lastHeaders = {};
  }

  async makeRequest(url, options = {}) {
    // Check if we're close to rate limit
    if (this.shouldWait()) {
      await this.waitForRateLimit();
    }

    const response = await fetch(url, {
      ...options,
      headers: {
        'X-API-Key': this.apiKey,
        'Content-Type': 'application/json',
        ...options.headers
      }
    });

    // Store rate limit headers
    this.lastHeaders = {
      limit: parseInt(response.headers.get('X-RateLimit-Limit')),
      remaining: parseInt(response.headers.get('X-RateLimit-Remaining')),
      reset: parseInt(response.headers.get('X-RateLimit-Reset')),
      burstRemaining: parseInt(response.headers.get('X-RateLimit-Burst-Remaining'))
    };

    if (response.status === 429) {
      const retryAfter = response.headers.get('Retry-After') || 60;
      throw new RateLimitError(`Rate limited. Retry after ${retryAfter} seconds`, retryAfter);
    }

    return response;
  }

  shouldWait() {
    const { remaining, burstRemaining } = this.lastHeaders;
    return remaining <= 5 || burstRemaining <= 2;
  }

  async waitForRateLimit() {
    const resetTime = this.lastHeaders.reset * 1000;
    const waitTime = Math.max(0, resetTime - Date.now()) + 1000; // Add 1 second buffer
    
    console.log(`Rate limit approaching. Waiting ${waitTime}ms...`);
    await new Promise(resolve => setTimeout(resolve, waitTime));
  }
}
```

### 2. Implement Exponential Backoff

```javascript
async function makeRequestWithBackoff(requestFn, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await requestFn();
    } catch (error) {
      if (error instanceof RateLimitError && attempt < maxRetries) {
        const delay = Math.min(1000 * Math.pow(2, attempt - 1), 60000); // Max 1 minute
        console.log(`Rate limited. Retrying in ${delay}ms (attempt ${attempt}/${maxRetries})`);
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
      throw error;
    }
  }
}

// Usage
const result = await makeRequestWithBackoff(async () => {
  return await client.calculateDrugs(patient, drugs);
});
```

### 3. Request Queuing

```javascript
class RequestQueue {
  constructor(rateLimitPerMinute = 100) {
    this.queue = [];
    this.processing = false;
    this.rateLimitPerMinute = rateLimitPerMinute;
    this.requestTimes = [];
  }

  async enqueue(requestFn) {
    return new Promise((resolve, reject) => {
      this.queue.push({ requestFn, resolve, reject });
      this.processQueue();
    });
  }

  async processQueue() {
    if (this.processing || this.queue.length === 0) {
      return;
    }

    this.processing = true;

    while (this.queue.length > 0) {
      // Clean old request times (older than 1 minute)
      const oneMinuteAgo = Date.now() - 60000;
      this.requestTimes = this.requestTimes.filter(time => time > oneMinuteAgo);

      // Check if we can make a request
      if (this.requestTimes.length >= this.rateLimitPerMinute) {
        const oldestRequest = Math.min(...this.requestTimes);
        const waitTime = oldestRequest + 60000 - Date.now();
        await new Promise(resolve => setTimeout(resolve, waitTime));
        continue;
      }

      const { requestFn, resolve, reject } = this.queue.shift();

      try {
        this.requestTimes.push(Date.now());
        const result = await requestFn();
        resolve(result);
      } catch (error) {
        reject(error);
      }
    }

    this.processing = false;
  }
}

// Usage
const queue = new RequestQueue(100); // 100 requests per minute

const result = await queue.enqueue(async () => {
  return await client.calculateDrugs(patient, drugs);
});
```

## Best Practices

### 1. Batch Operations

Instead of making multiple individual requests, batch them when possible:

```javascript
// ❌ Poor: Multiple individual requests
const results = [];
for (const patient of patients) {
  const result = await client.calculateDrugs(patient, drugs);
  results.push(result);
}

// ✅ Better: Batch processing with rate limit awareness
const results = [];
const batchSize = 10;

for (let i = 0; i < patients.length; i += batchSize) {
  const batch = patients.slice(i, i + batchSize);
  const batchPromises = batch.map(patient => 
    client.calculateDrugs(patient, drugs)
  );
  
  const batchResults = await Promise.all(batchPromises);
  results.push(...batchResults);
  
  // Small delay between batches
  if (i + batchSize < patients.length) {
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
}
```

### 2. Cache Frequently Accessed Data

```javascript
class CachedVetDrugsClient {
  constructor(apiKey, cacheTtl = 300000) { // 5 minutes
    this.client = new VetDrugsApiClient(apiKey);
    this.cache = new Map();
    this.cacheTtl = cacheTtl;
  }

  async searchDrugs(searchTerm, category) {
    const cacheKey = `search:${searchTerm}:${category}`;
    const cached = this.getFromCache(cacheKey);
    
    if (cached) {
      return cached;
    }

    const result = await this.client.searchDrugs(searchTerm, category);
    this.setCache(cacheKey, result);
    return result;
  }

  getFromCache(key) {
    const item = this.cache.get(key);
    if (item && Date.now() - item.timestamp < this.cacheTtl) {
      return item.data;
    }
    this.cache.delete(key);
    return null;
  }

  setCache(key, data) {
    this.cache.set(key, {
      data,
      timestamp: Date.now()
    });
  }
}
```

### 3. Monitor Rate Limit Usage

```javascript
class RateLimitMonitor {
  constructor() {
    this.stats = {
      requests: 0,
      rateLimited: 0,
      lastReset: Date.now()
    };
  }

  recordRequest(headers) {
    this.stats.requests++;
    
    const limit = parseInt(headers['x-ratelimit-limit']);
    const remaining = parseInt(headers['x-ratelimit-remaining']);
    const reset = parseInt(headers['x-ratelimit-reset']) * 1000;

    // Calculate usage percentage
    const usagePercent = ((limit - remaining) / limit) * 100;
    
    if (usagePercent > 80) {
      console.warn(`High rate limit usage: ${usagePercent.toFixed(1)}%`);
    }

    // Reset stats if rate limit window has reset
    if (reset > this.stats.lastReset) {
      this.stats.lastReset = reset;
      this.stats.requests = 0;
    }
  }

  recordRateLimit() {
    this.stats.rateLimited++;
    console.error(`Rate limited! Total rate limits: ${this.stats.rateLimited}`);
  }

  getStats() {
    return { ...this.stats };
  }
}
```

## Language-Specific Examples

### Python with Rate Limiting

```python
import time
import threading
from collections import deque

class RateLimitedVetDrugsClient:
    def __init__(self, api_key, rate_limit=100):
        self.client = VetDrugsApiClient(api_key)
        self.rate_limit = rate_limit
        self.requests = deque()
        self.lock = threading.Lock()

    def _wait_if_needed(self):
        with self.lock:
            now = time.time()
            # Remove requests older than 1 minute
            while self.requests and self.requests[0] <= now - 60:
                self.requests.popleft()
            
            # Check if we need to wait
            if len(self.requests) >= self.rate_limit:
                sleep_time = self.requests[0] + 60 - now
                if sleep_time > 0:
                    time.sleep(sleep_time)
            
            # Record this request
            self.requests.append(now)

    def calculate_drugs(self, patient, drugs):
        self._wait_if_needed()
        return self.client.calculate_drugs(patient, drugs)
```

### C# with Rate Limiting

```csharp
public class RateLimitedVetDrugsClient : IDisposable
{
    private readonly VetDrugsApiClient _client;
    private readonly SemaphoreSlim _semaphore;
    private readonly Queue<DateTime> _requestTimes;
    private readonly object _lockObject = new object();
    private readonly int _rateLimitPerMinute;

    public RateLimitedVetDrugsClient(string apiKey, int rateLimitPerMinute = 100)
    {
        _client = new VetDrugsApiClient(apiKey);
        _rateLimitPerMinute = rateLimitPerMinute;
        _semaphore = new SemaphoreSlim(rateLimitPerMinute, rateLimitPerMinute);
        _requestTimes = new Queue<DateTime>();
    }

    public async Task<CalculationResponse> CalculateDrugsAsync(CalculationRequest request)
    {
        await WaitForRateLimit();
        
        try
        {
            return await _client.CalculateDrugsAsync(request);
        }
        finally
        {
            RecordRequest();
        }
    }

    private async Task WaitForRateLimit()
    {
        lock (_lockObject)
        {
            var oneMinuteAgo = DateTime.Now.AddMinutes(-1);
            while (_requestTimes.Count > 0 && _requestTimes.Peek() < oneMinuteAgo)
            {
                _requestTimes.Dequeue();
            }

            if (_requestTimes.Count >= _rateLimitPerMinute)
            {
                var waitTime = _requestTimes.Peek().AddMinutes(1) - DateTime.Now;
                if (waitTime > TimeSpan.Zero)
                {
                    Thread.Sleep(waitTime);
                }
            }
        }
    }

    private void RecordRequest()
    {
        lock (_lockObject)
        {
            _requestTimes.Enqueue(DateTime.Now);
        }
    }

    public void Dispose()
    {
        _client?.Dispose();
        _semaphore?.Dispose();
    }
}
```

### PHP with Rate Limiting

```php
class RateLimitedVetDrugsClient
{
    private $client;
    private $rateLimitPerMinute;
    private $requestTimes = [];

    public function __construct($apiKey, $rateLimitPerMinute = 100)
    {
        $this->client = new VetDrugsApiClient($apiKey);
        $this->rateLimitPerMinute = $rateLimitPerMinute;
    }

    public function calculateDrugs($patient, $drugs)
    {
        $this->waitIfNeeded();
        $this->recordRequest();
        
        return $this->client->calculateDrugs($patient, $drugs);
    }

    private function waitIfNeeded()
    {
        $now = time();
        $oneMinuteAgo = $now - 60;
        
        // Remove old requests
        $this->requestTimes = array_filter($this->requestTimes, function($time) use ($oneMinuteAgo) {
            return $time > $oneMinuteAgo;
        });
        
        // Check if we need to wait
        if (count($this->requestTimes) >= $this->rateLimitPerMinute) {
            $oldestRequest = min($this->requestTimes);
            $waitTime = ($oldestRequest + 60) - $now;
            
            if ($waitTime > 0) {
                sleep($waitTime);
            }
        }
    }

    private function recordRequest()
    {
        $this->requestTimes[] = time();
    }
}
```

## Rate Limit Optimization Strategies

### 1. Prioritize Critical Requests

```javascript
class PrioritizedRequestQueue {
  constructor() {
    this.highPriority = [];
    this.normalPriority = [];
    this.lowPriority = [];
  }

  enqueue(requestFn, priority = 'normal') {
    const request = { requestFn, promise: null };
    
    request.promise = new Promise((resolve, reject) => {
      request.resolve = resolve;
      request.reject = reject;
    });

    switch (priority) {
      case 'high':
        this.highPriority.push(request);
        break;
      case 'low':
        this.lowPriority.push(request);
        break;
      default:
        this.normalPriority.push(request);
    }

    this.processNext();
    return request.promise;
  }

  processNext() {
    let request;
    
    if (this.highPriority.length > 0) {
      request = this.highPriority.shift();
    } else if (this.normalPriority.length > 0) {
      request = this.normalPriority.shift();
    } else if (this.lowPriority.length > 0) {
      request = this.lowPriority.shift();
    }

    if (request) {
      this.executeRequest(request);
    }
  }

  async executeRequest(request) {
    try {
      const result = await request.requestFn();
      request.resolve(result);
    } catch (error) {
      request.reject(error);
    }
    
    // Process next request after a small delay
    setTimeout(() => this.processNext(), 100);
  }
}
```

### 2. Intelligent Request Merging

```javascript
class RequestMerger {
  constructor() {
    this.pendingRequests = new Map();
  }

  async searchDrugs(searchTerm, category) {
    const key = `${searchTerm}:${category}`;
    
    if (this.pendingRequests.has(key)) {
      // Return the existing promise
      return this.pendingRequests.get(key);
    }

    const promise = this.client.searchDrugs(searchTerm, category);
    this.pendingRequests.set(key, promise);

    try {
      const result = await promise;
      return result;
    } finally {
      // Clean up after request completes
      setTimeout(() => {
        this.pendingRequests.delete(key);
      }, 1000);
    }
  }
}
```

## Monitoring and Alerting

### Rate Limit Dashboard

```javascript
class RateLimitDashboard {
  constructor() {
    this.metrics = {
      requestsPerMinute: 0,
      requestsPerHour: 0,
      rateLimitHits: 0,
      averageResponseTime: 0
    };
    
    this.startMonitoring();
  }

  startMonitoring() {
    setInterval(() => {
      this.updateDashboard();
    }, 60000); // Update every minute
  }

  recordRequest(responseTime, wasRateLimited = false) {
    this.metrics.requestsPerMinute++;
    this.metrics.requestsPerHour++;
    
    if (wasRateLimited) {
      this.metrics.rateLimitHits++;
    }

    // Update average response time
    this.updateAverageResponseTime(responseTime);
  }

  updateDashboard() {
    console.log('VetDrugs API Usage:', this.metrics);
    
    // Alert if usage is high
    if (this.metrics.requestsPerMinute > 80) {
      console.warn('High API usage detected!');
    }

    // Reset per-minute counter
    this.metrics.requestsPerMinute = 0;
  }

  updateAverageResponseTime(responseTime) {
    if (this.metrics.averageResponseTime === 0) {
      this.metrics.averageResponseTime = responseTime;
    } else {
      this.metrics.averageResponseTime = 
        (this.metrics.averageResponseTime * 0.9) + (responseTime * 0.1);
    }
  }
}
```

## Next Steps

- [Review error handling](errors.md)
- [Explore API endpoints](calculate.md)
- [Check integration guides](../integrations/)
- [Contact support](../support.md) for rate limit increases
