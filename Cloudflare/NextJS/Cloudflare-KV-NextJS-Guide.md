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