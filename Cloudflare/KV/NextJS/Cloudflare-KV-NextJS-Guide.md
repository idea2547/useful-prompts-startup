# Cloudflare KV + Next.js App Router: The Working Solution
**🚀 Production-Ready Integration Guide**

## TL;DR - The Solution That Actually Works

After extensive testing, here's the **exact pattern** that works for Cloudflare KV with Next.js App Router on Cloudflare Pages:

```typescript
// ✅ CORRECT: Access KV from process.env
const KV_STORE = (process.env as any).YOUR_KV_BINDING_NAME

// ❌ WRONG: These don't work on Cloudflare Pages
const KV_STORE = context?.env?.YOUR_KV_BINDING_NAME
const KV_STORE = (globalThis as any).YOUR_KV_BINDING_NAME
```

## 🎯 Key Discovery

**Cloudflare Pages exposes KV bindings through `process.env`, not context parameters!**

This is the critical insight that most tutorials miss.

## Quick Setup

### 1. Create KV Namespace
```bash
# Via Wrangler CLI
wrangler kv:namespace create YOUR_NAMESPACE_NAME

# Or via Cloudflare Dashboard:
# Workers & Pages → KV → Create namespace
```

### 2. Configure Binding
**Cloudflare Dashboard** → **Your Pages Project** → **Settings** → **Functions** → **KV namespace bindings**

Add binding:
- **Variable name**: `YOUR_KV_BINDING`
- **KV namespace**: `YOUR_NAMESPACE_NAME`

### 3. API Route Implementation

```typescript
// app/api/data/route.ts
import { NextRequest, NextResponse } from 'next/server'

export const runtime = 'edge' // Required!

async function getKVData(): Promise<any> {
  try {
    // 🔑 KEY: Use process.env for Cloudflare Pages
    const KV_STORE = (process.env as any).YOUR_KV_BINDING
    
    if (KV_STORE) {
      console.log('✅ KV connected')
      const data = await KV_STORE.get('your-key')
      return data ? JSON.parse(data) : null
    } else {
      console.log('❌ KV not available')
      return null
    }
  } catch (error) {
    console.error('KV error:', error)
    return null
  }
}

async function saveKVData(data: any): Promise<boolean> {
  try {
    const KV_STORE = (process.env as any).YOUR_KV_BINDING
    
    if (KV_STORE) {
      await KV_STORE.put('your-key', JSON.stringify(data))
      return true
    }
    return false
  } catch (error) {
    console.error('Save error:', error)
    return false
  }
}

export async function GET() {
  const data = await getKVData()
  return NextResponse.json({ success: !!data, data })
}

export async function POST(request: NextRequest) {
  const body = await request.json()
  const success = await saveKVData(body)
  return NextResponse.json({ success })
}
```

### 4. Test Endpoint

```typescript
// app/api/test-kv/route.ts
import { NextResponse } from 'next/server'

export const runtime = 'edge'

export async function GET() {
  try {
    const KV_STORE = (process.env as any).YOUR_KV_BINDING
    
    if (KV_STORE) {
      // Test write
      await KV_STORE.put('test', JSON.stringify({ 
        message: 'KV is working!', 
        timestamp: Date.now() 
      }))
      
      // Test read
      const result = await KV_STORE.get('test')
      
      return NextResponse.json({
        success: true,
        message: '🎉 KV is working!',
        data: result ? JSON.parse(result) : null
      })
    }
    
    return NextResponse.json({
      success: false,
      message: 'KV binding not found'
    })
  } catch (error) {
    return NextResponse.json({
      success: false,
      error: error instanceof Error ? error.message : 'Unknown error'
    }, { status: 500 })
  }
}
```

## React Hook for KV Data

```typescript
// hooks/useKVData.ts
'use client'
import { useState, useEffect } from 'react'

export function useKVData<T>(key: string, initialData: T) {
  const [data, setData] = useState<T>(initialData)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    fetchData()
  }, [key])

  const fetchData = async () => {
    try {
      const response = await fetch('/api/data')
      const result = await response.json()
      
      if (result.success && result.data) {
        setData(result.data)
      }
    } catch (error) {
      console.error('Fetch error:', error)
    } finally {
      setLoading(false)
    }
  }

  const updateData = async (newData: T) => {
    try {
      const response = await fetch('/api/data', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newData)
      })
      
      const result = await response.json()
      
      if (result.success) {
        setData(newData)
        return true
      }
    } catch (error) {
      console.error('Update error:', error)
    }
    return false
  }

  return { data, loading, updateData, refetch: fetchData }
}
```

## Analytics Dashboard Example

### Complete Dashboard Page
```typescript
// app/dashboard/page.tsx
'use client'
import { useState, useEffect } from 'react'

interface AnalyticsData {
  [key: string]: number
}

interface DashboardMetric {
  id: string
  label: string
  description: string
  value: number
  icon: string
  color: string
}

export default function AnalyticsDashboard() {
  const [data, setData] = useState<AnalyticsData>({})
  const [isLoading, setIsLoading] = useState(true)
  const [lastUpdated, setLastUpdated] = useState<Date | null>(null)

  useEffect(() => {
    fetchData()
    // Auto-refresh every 10 seconds
    const interval = setInterval(fetchData, 10000)
    return () => clearInterval(interval)
  }, [])

  const fetchData = async () => {
    try {
      const response = await fetch('/api/analytics')
      const result = await response.json()
      
      if (result.success) {
        setData(result.data)
        setLastUpdated(new Date())
      }
    } catch (error) {
      console.error('Error fetching analytics:', error)
    } finally {
      setIsLoading(false)
    }
  }

  // Define your metrics with descriptions
  const metricDefinitions: Record<string, Omit<DashboardMetric, 'value'>> = {
    'button-clicks': {
      id: 'button-clicks',
      label: 'Button Clicks',
      description: 'Total button interactions',
      icon: '👆',
      color: 'bg-blue-500'
    },
    'page-views': {
      id: 'page-views',
      label: 'Page Views',
      description: 'Unique page visits',
      icon: '👀',
      color: 'bg-green-500'
    },
    'feature-interactions': {
      id: 'feature-interactions',
      label: 'Feature Usage',
      description: 'Feature engagement count',
      icon: '⚡',
      color: 'bg-purple-500'
    },
    'user-signups': {
      id: 'user-signups',
      label: 'Sign-ups',
      description: 'New user registrations',
      icon: '🎉',
      color: 'bg-orange-500'
    }
  }

  // Transform data into metrics
  const metrics: DashboardMetric[] = Object.entries(data).map(([key, value]) => ({
    ...metricDefinitions[key] || {
      id: key,
      label: key.replace(/-/g, ' ').replace(/\b\w/g, l => l.toUpperCase()),
      description: `Total ${key.replace(/-/g, ' ')} count`,
      icon: '📊',
      color: 'bg-gray-500'
    },
    value
  }))

  const totalInteractions = Object.values(data).reduce((sum, count) => sum + count, 0)

  if (isLoading) {
    return (
      <div className="min-h-screen bg-gray-50 flex items-center justify-center">
        <div className="text-center">
          <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-600 mx-auto mb-4"></div>
          <p className="text-gray-600">Loading analytics...</p>
        </div>
      </div>
    )
  }

  return (
    <div className="min-h-screen bg-gray-50 p-4 md:p-8">
      <div className="max-w-7xl mx-auto">
        {/* Header */}
        <div className="mb-8">
          <h1 className="text-3xl md:text-4xl font-bold text-gray-900 mb-2">
            📊 Analytics Dashboard
          </h1>
          <p className="text-gray-600">
            Real-time insights powered by Cloudflare KV
          </p>
          {lastUpdated && (
            <p className="text-sm text-gray-500 mt-2">
              Last updated: {lastUpdated.toLocaleTimeString()}
            </p>
          )}
        </div>

        {/* Summary Card */}
        <div className="bg-white rounded-lg shadow-sm border border-gray-200 p-6 mb-8">
          <div className="text-center">
            <h2 className="text-lg font-semibold text-gray-700 mb-2">
              Total Interactions
            </h2>
            <p className="text-4xl md:text-5xl font-bold text-blue-600">
              {totalInteractions.toLocaleString()}
            </p>
            <p className="text-gray-500 mt-2">
              Across all tracked events
            </p>
          </div>
        </div>

        {/* Metrics Grid */}
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6 mb-8">
          {metrics.map((metric) => (
            <MetricCard key={metric.id} metric={metric} />
          ))}
        </div>

        {/* Detailed Breakdown */}
        <div className="bg-white rounded-lg shadow-sm border border-gray-200">
          <div className="p-6 border-b border-gray-200">
            <h3 className="text-lg font-semibold text-gray-900">
              Detailed Breakdown
            </h3>
          </div>
          <div className="p-6">
            {metrics.length === 0 ? (
              <p className="text-gray-500 text-center py-8">
                No data available yet. Start tracking events to see analytics!
              </p>
            ) : (
              <div className="space-y-4">
                {metrics
                  .sort((a, b) => b.value - a.value)
                  .map((metric) => (
                    <MetricRow key={metric.id} metric={metric} total={totalInteractions} />
                  ))}
              </div>
            )}
          </div>
        </div>

        {/* Footer */}
        <div className="mt-8 text-center text-sm text-gray-500">
          <p>🔄 Auto-refreshes every 10 seconds</p>
          <p>💾 Data stored in Cloudflare KV</p>
        </div>
      </div>
    </div>
  )
}

// Metric Card Component
interface MetricCardProps {
  metric: DashboardMetric
}

function MetricCard({ metric }: MetricCardProps) {
  return (
    <div className="bg-white rounded-lg shadow-sm border border-gray-200 p-6 hover:shadow-md transition-shadow">
      <div className="flex items-center justify-between mb-4">
        <div className={`w-12 h-12 rounded-lg ${metric.color} flex items-center justify-center text-white text-xl`}>
          {metric.icon}
        </div>
        <div className="text-right">
          <p className="text-2xl font-bold text-gray-900">
            {metric.value.toLocaleString()}
          </p>
        </div>
      </div>
      <h4 className="font-semibold text-gray-900 mb-1">
        {metric.label}
      </h4>
      <p className="text-sm text-gray-600">
        {metric.description}
      </p>
    </div>
  )
}

// Metric Row Component
interface MetricRowProps {
  metric: DashboardMetric
  total: number
}

function MetricRow({ metric, total }: MetricRowProps) {
  const percentage = total > 0 ? (metric.value / total) * 100 : 0

  return (
    <div className="flex items-center justify-between p-4 bg-gray-50 rounded-lg">
      <div className="flex items-center space-x-4">
        <div className={`w-10 h-10 rounded-full ${metric.color} flex items-center justify-center text-white`}>
          {metric.icon}
        </div>
        <div>
          <h4 className="font-medium text-gray-900">{metric.label}</h4>
          <p className="text-sm text-gray-600">{metric.description}</p>
        </div>
      </div>
      <div className="text-right">
        <p className="text-lg font-semibold text-gray-900">
          {metric.value.toLocaleString()}
        </p>
        <p className="text-sm text-gray-500">
          {percentage.toFixed(1)}% of total
        </p>
      </div>
    </div>
  )
}
```

### Analytics API Route
```typescript
// app/api/analytics/route.ts
import { NextResponse } from 'next/server'

export const runtime = 'edge'

interface AnalyticsData {
  [key: string]: number
}

// Default analytics structure
const defaultAnalytics: AnalyticsData = {
  'button-clicks': 0,
  'page-views': 0,
  'feature-interactions': 0,
  'user-signups': 0
}

async function getAnalyticsData(): Promise<AnalyticsData> {
  try {
    const ANALYTICS_KV = (process.env as any).ANALYTICS_KV
    
    if (ANALYTICS_KV) {
      const stored = await ANALYTICS_KV.get('analytics-data')
      
      if (stored) {
        const parsed = JSON.parse(stored)
        // Merge with defaults to ensure all keys exist
        return { ...defaultAnalytics, ...parsed }
      } else {
        // Initialize with default data
        await ANALYTICS_KV.put('analytics-data', JSON.stringify(defaultAnalytics))
        return { ...defaultAnalytics }
      }
    }
    
    return { ...defaultAnalytics }
  } catch (error) {
    console.error('Error fetching analytics:', error)
    return { ...defaultAnalytics }
  }
}

async function incrementMetric(metric: string): Promise<AnalyticsData> {
  try {
    const ANALYTICS_KV = (process.env as any).ANALYTICS_KV
    
    if (ANALYTICS_KV) {
      const current = await getAnalyticsData()
      
      if (metric in current) {
        current[metric]++
      } else {
        current[metric] = 1
      }
      
      await ANALYTICS_KV.put('analytics-data', JSON.stringify(current))
      return current
    }
    
    return defaultAnalytics
  } catch (error) {
    console.error('Error incrementing metric:', error)
    return defaultAnalytics
  }
}

// GET: Fetch analytics data
export async function GET() {
  try {
    const data = await getAnalyticsData()
    return NextResponse.json({ 
      success: true, 
      data,
      timestamp: new Date().toISOString()
    })
  } catch (error) {
    return NextResponse.json(
      { success: false, error: 'Failed to fetch analytics' },
      { status: 500 }
    )
  }
}

// POST: Track new event
export async function POST(request: Request) {
  try {
    const { event, metadata } = await request.json()
    
    if (!event || typeof event !== 'string') {
      return NextResponse.json(
        { success: false, error: 'Event name is required' },
        { status: 400 }
      )
    }

    // Validate event name (optional security measure)
    const allowedEvents = [
      'button-clicks',
      'page-views', 
      'feature-interactions',
      'user-signups'
    ]
    
    if (!allowedEvents.includes(event)) {
      return NextResponse.json(
        { success: false, error: 'Invalid event type' },
        { status: 400 }
      )
    }

    const updatedData = await incrementMetric(event)

    console.log(`📈 Analytics: ${event} incremented to ${updatedData[event]}`)

    return NextResponse.json({ 
      success: true, 
      event,
      count: updatedData[event],
      data: updatedData
    })
  } catch (error) {
    console.error('Analytics tracking error:', error)
    return NextResponse.json(
      { success: false, error: 'Failed to track event' },
      { status: 500 }
    )
  }
}
```

### Event Tracking Hook
```typescript
// hooks/useAnalytics.ts
'use client'
import { useCallback } from 'react'

export function useAnalytics() {
  const trackEvent = useCallback(async (event: string, metadata?: any) => {
    try {
      const response = await fetch('/api/analytics', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ event, metadata })
      })

      const result = await response.json()
      
      if (result.success) {
        console.log(`✅ Tracked: ${event}`)
        return result
      } else {
        console.error('Failed to track event:', result.error)
      }
    } catch (error) {
      console.error('Analytics error:', error)
    }
  }, [])

  return { trackEvent }
}
```

### Usage in Components
```typescript
// components/TrackableButton.tsx
'use client'
import { useAnalytics } from '@/hooks/useAnalytics'

interface TrackableButtonProps {
  children: React.ReactNode
  eventName: string
  className?: string
  onClick?: () => void
}

export function TrackableButton({ 
  children, 
  eventName, 
  className = '', 
  onClick 
}: TrackableButtonProps) {
  const { trackEvent } = useAnalytics()

  const handleClick = () => {
    // Track the event
    trackEvent(eventName)
    
    // Execute additional onClick if provided
    onClick?.()
  }

  return (
    <button 
      onClick={handleClick}
      className={`px-4 py-2 bg-blue-600 text-white rounded hover:bg-blue-700 transition-colors ${className}`}
    >
      {children}
    </button>
  )
}

// Usage example:
// <TrackableButton eventName="button-clicks">
//   Click me!
// </TrackableButton>
```

## Common Pitfalls to Avoid

### ❌ Wrong Patterns
```typescript
// These DON'T work on Cloudflare Pages
export async function GET(request: NextRequest, context: any) {
  const KV = context?.env?.YOUR_KV_BINDING // ❌ Won't work
  const KV = (globalThis as any).YOUR_KV_BINDING // ❌ Won't work
}
```

### ✅ Correct Pattern
```typescript
// This DOES work on Cloudflare Pages
export async function GET() {
  const KV = (process.env as any).YOUR_KV_BINDING // ✅ Works!
}
```

### Other Requirements
```typescript
// Always required for Cloudflare Pages
export const runtime = 'edge'
```

## Troubleshooting Checklist

1. **Binding Configuration**
   - Variable name matches exactly in code
   - Namespace exists and is spelled correctly
   - Redeploy after binding changes

2. **Code Requirements**
   - API routes must include `export const runtime = 'edge'`
   - Use `(process.env as any).YOUR_BINDING_NAME`
   - Handle KV unavailability gracefully

3. **Testing**
   - Test `/api/test-kv` endpoint first
   - Check Cloudflare Functions logs
   - Verify in Cloudflare KV dashboard

## Environment Setup

### Development
```typescript
// For local development, add fallback
const KV_STORE = (process.env as any).YOUR_KV_BINDING || null

if (!KV_STORE) {
  // Use local storage or in-memory fallback
  console.log('Using development fallback')
}
```

### Production
- Ensure KV namespace exists
- Verify binding configuration
- Test after deployment

## Performance Tips

### KV Limits (Cloudflare)
- **Free**: 100K reads, 1K writes per day
- **Paid**: Unlimited reads, 1M+ writes per month

### Optimization
```typescript
// Cache KV data when possible
const CACHE_TTL = 300 // 5 minutes

export async function GET() {
  const cached = await KV_STORE.get('cached-data')
  
  if (cached) {
    const { data, timestamp } = JSON.parse(cached)
    if (Date.now() - timestamp < CACHE_TTL * 1000) {
      return NextResponse.json({ data, fromCache: true })
    }
  }
  
  // Fetch fresh data and cache it
  const freshData = await fetchFreshData()
  await KV_STORE.put('cached-data', JSON.stringify({
    data: freshData,
    timestamp: Date.now()
  }))
  
  return NextResponse.json({ data: freshData, fromCache: false })
}
```

## Security Best Practices

### Input Validation
```typescript
const allowedKeys = ['user-data', 'app-config', 'analytics']

export async function POST(request: NextRequest) {
  const { key, data } = await request.json()
  
  if (!allowedKeys.includes(key)) {
    return NextResponse.json(
      { error: 'Invalid key' }, 
      { status: 400 }
    )
  }
  
  // Process valid request
}
```

### Rate Limiting
```typescript
// Simple rate limiting by IP
const rateLimits = new Map()

export async function POST(request: NextRequest) {
  const ip = request.headers.get('cf-connecting-ip') || 'unknown'
  const now = Date.now()
  const windowMs = 60 * 1000 // 1 minute
  
  const requests = rateLimits.get(ip) || []
  const recentRequests = requests.filter((time: number) => now - time < windowMs)
  
  if (recentRequests.length >= 10) { // Max 10 requests per minute
    return NextResponse.json(
      { error: 'Rate limit exceeded' }, 
      { status: 429 }
    )
  }
  
  recentRequests.push(now)
  rateLimits.set(ip, recentRequests)
  
  // Process request
}
```

## Real-World Examples

### Counter App
```typescript
// Increment counter
export async function POST() {
  const KV = (process.env as any).COUNTER_KV
  const current = await KV.get('count') || '0'
  const newCount = parseInt(current) + 1
  await KV.put('count', newCount.toString())
  return NextResponse.json({ count: newCount })
}
```

### User Preferences
```typescript
// Save user settings
export async function POST(request: NextRequest) {
  const { userId, preferences } = await request.json()
  const KV = (process.env as any).USER_DATA_KV
  
  await KV.put(`user:${userId}:prefs`, JSON.stringify(preferences))
  return NextResponse.json({ success: true })
}
```

### Analytics Tracking
```typescript
// Track page views
export async function POST(request: NextRequest) {
  const { page } = await request.json()
  const KV = (process.env as any).ANALYTICS_KV
  
  const key = `views:${page}:${new Date().toISOString().split('T')[0]}`
  const current = await KV.get(key) || '0'
  await KV.put(key, (parseInt(current) + 1).toString())
  
  return NextResponse.json({ success: true })
}
```

## Why This Works

**The Problem**: Most tutorials assume Cloudflare Workers context patterns work on Cloudflare Pages.

**The Reality**: Cloudflare Pages has a different runtime environment where KV bindings are exposed through `process.env`.

**The Solution**: Use `(process.env as any).YOUR_BINDING_NAME` instead of context-based access.

## Resources

- [Cloudflare KV Docs](https://developers.cloudflare.com/workers/runtime-apis/kv/)
- [Cloudflare Pages Functions](https://developers.cloudflare.com/pages/platform/functions/)
- [Next.js App Router](https://nextjs.org/docs/app)

---

**Found this helpful?** Share it to help other developers! 🚀

**Having issues?** The key is always `process.env` access pattern for Cloudflare Pages.

---

*Last updated: December 2024* 