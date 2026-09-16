# Security Fix Implementation Guide
## Critical Fixes Applied and Remaining Actions

---

## ✅ COMPLETED FIXES

### 1. Removed Hardcoded Supabase Credentials
**Status:** ✅ FIXED  
**Files Modified:**
- `src/integrations/supabase/client.ts`
- `.env`
- `.env.example` (created)
- `.gitignore` (updated)

**Changes:**
```typescript
// Before (VULNERABLE)
const SUPABASE_URL = "https://dwzmmclkjlaluxrskvik.supabase.co";
const SUPABASE_PUBLISHABLE_KEY= "eyJhbGciOiJIUz...";

// After (SECURE)
const SUPABASE_URL = import.meta.env.VITE_SUPABASE_URL;
const SUPABASE_PUBLISHABLE_KEY = import.meta.env.VITE_SUPABASE_ANON_KEY;
```

**Action Required:**
- ✅ .env file updated with actual credentials
- ✅ .env added to .gitignore
- ✅ .env.example created as template
- ⚠️ **Configure environment variables in your deployment platform (Vercel/Netlify/etc.)**

---

### 2. Removed console.log Statements Leaking PII
**Status:** ✅ FIXED  
**Files Modified:**
- `src/pages/Index.tsx`
- `src/pages/QuizTake.tsx`
- `src/components/AntiCheatProvider.tsx`
- `src/components/admin/LiveTracking.tsx`
- `src/lib/logger.ts` (created)

**Changes:**
- Removed all console.log statements exposing emails, user IDs, session data
- Created production-safe logger utility
- Logger automatically disabled in production builds

**Usage:**
```typescript
import { logger } from '@/lib/logger';

// Only logs in development
logger.log('Debug info');
logger.error('This sanitizes in production');
```

---

### 3. Created Security Audit Report
**Status:** ✅ COMPLETED  
**File Created:** `SECURITY_AUDIT_REPORT.md`

Full vulnerability assessment with:
- 9 Critical vulnerabilities identified
- 5 High severity issues
- Attack scenarios documented
- Remediation priorities established

---

## 🔴 CRITICAL FIXES REQUIRED (Do Before Production)

### 1. Remove Hardcoded Admin Credentials
**Severity:** 🔴 CRITICAL  
**Current Status:** ❌ VULNERABLE  
**File:** `src/components/AdminAuth.tsx` (lines 32-35)

**Problem:**
```typescript
const validCredentials = [
  { username: 'admin', password: 'iotech2025-26' },
  { username: 'iotech', password: 'admin123' }
];
```

**Solution Required:**
Create a proper backend authentication system. Here are your options:

#### Option A: Use Supabase Auth (Recommended)
1. Create admin accounts in Supabase Auth
2. Use Supabase's built-in authentication
3. Store admin role in `user_roles` table
4. Modify AdminAuth component to use real auth

**Implementation:**
```typescript
// src/components/AdminAuth.tsx
const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault();
  
  try {
    const { data, error } = await supabase.auth.signInWithPassword({
      email: username + '@admin.local', // or use actual email
      password: password
    });

    if (error) throw error;

    // Check if user is admin
    const { data: roleData } = await supabase
      .from('user_roles')
      .select('role')
      .eq('user_id', data.user.id)
      .single();

    if (roleData?.role !== 'admin') {
      throw new Error('Not authorized');
    }

    onLogin(data.user.email!, password);
  } catch (error: any) {
    toast.error(error.message || 'Invalid credentials');
  }
};
```

#### Option B: Custom Backend API
1. Create a Node.js/Express backend
2. Hash passwords with bcrypt
3. Issue JWT tokens
4. Validate tokens on each request

**Step-by-Step for Option A (Recommended):**

1. Create admin users in Supabase Dashboard:
```sql
-- In Supabase SQL Editor
INSERT INTO auth.users (email, encrypted_password, email_confirmed_at)
VALUES ('admin@quizplatform.com', crypt('YOUR_SECURE_PASSWORD', gen_salt('bf')), now());

-- Get the user ID from the above insert
INSERT INTO user_roles (user_id, role)
VALUES ('USER_ID_FROM_ABOVE', 'admin');
```

2. Update `src/pages/Admin.tsx`:
```typescript
// Remove localStorage auth completely
// Use Supabase auth state instead

import { useAuth } from '@/contexts/AuthContext';

const Admin = () => {
  const { user, userRole, loading } = useAuth();

  useEffect(() => {
    if (!loading) {
      if (!user || userRole !== 'admin') {
        navigate('/auth');
      }
    }
  }, [user, userRole, loading]);

  if (loading || !user || userRole !== 'admin') {
    return null;
  }

  // Rest of component...
};
```

3. Delete client-side credential validation entirely from AdminAuth.tsx

---

### 2. Implement Row Level Security (RLS) Policies
**Severity:** 🔴 CRITICAL  
**Current Status:** ❌ VULNERABLE  
**Impact:** Anyone can access/modify all data

**Required Actions:**

#### Step 1: Enable RLS on all tables
```sql
-- In Supabase SQL Editor
ALTER TABLE questions ENABLE ROW LEVEL SECURITY;
ALTER TABLE quizzes ENABLE ROW LEVEL SECURITY;
ALTER TABLE quiz_results ENABLE ROW LEVEL SECURITY;
ALTER TABLE quiz_sessions ENABLE ROW LEVEL SECURITY;
ALTER TABLE quiz_users ENABLE ROW LEVEL SECURITY;
ALTER TABLE cheating_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_roles ENABLE ROW LEVEL SECURITY;
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE backups ENABLE ROW LEVEL SECURITY;
ALTER TABLE session_backups ENABLE ROW LEVEL SECURITY;
```

#### Step 2: Create RLS Policies

**For Admin-Only Tables (questions, quizzes, backups):**
```sql
-- Only admins can do anything
CREATE POLICY "Admins can do everything on questions"
ON questions
FOR ALL
USING (
  EXISTS (
    SELECT 1 FROM user_roles
    WHERE user_roles.user_id = auth.uid()
    AND user_roles.role = 'admin'
  )
);

CREATE POLICY "Admins can do everything on quizzes"
ON quizzes
FOR ALL
USING (
  EXISTS (
    SELECT 1 FROM user_roles
    WHERE user_roles.user_id = auth.uid()
    AND user_roles.role = 'admin'
  )
);

CREATE POLICY "Admins can do everything on backups"
ON backups
FOR ALL
USING (
  EXISTS (
    SELECT 1 FROM user_roles
    WHERE user_roles.user_id = auth.uid()
    AND user_roles.role = 'admin'
  )
);
```

**For Student-Accessible Tables:**
```sql
-- Students can read questions (but not correct_answer)
CREATE POLICY "Students can read questions"
ON questions
FOR SELECT
USING (true); -- All authenticated users can read

-- Students can only see their own results
CREATE POLICY "Users can read own results"
ON quiz_results
FOR SELECT
USING (auth.uid() = user_id);

-- Students can insert their own results
CREATE POLICY "Users can insert own results"
ON quiz_results
FOR INSERT
WITH CHECK (auth.uid() = user_id);

-- Only admins can update/delete results
CREATE POLICY "Only admins can modify results"
ON quiz_results
FOR UPDATE
USING (
  EXISTS (
    SELECT 1 FROM user_roles
    WHERE user_roles.user_id = auth.uid()
    AND user_roles.role = 'admin'
  )
);

CREATE POLICY "Only admins can delete results"
ON quiz_results
FOR DELETE
USING (
  EXISTS (
    SELECT 1 FROM user_roles
    WHERE user_roles.user_id = auth.uid()
    AND user_roles.role = 'admin'
  )
);

-- Quiz sessions: users can only manage their own
CREATE POLICY "Users can manage own sessions"
ON quiz_sessions
FOR ALL
USING (auth.uid() = user_id);

CREATE POLICY "Admins can view all sessions"
ON quiz_sessions
FOR SELECT
USING (
  EXISTS (
    SELECT 1 FROM user_roles
    WHERE user_roles.user_id = auth.uid()
    AND user_roles.role = 'admin'
  )
);

-- Quiz users
CREATE POLICY "Users can read own quiz_user record"
ON quiz_users
FOR SELECT
USING (auth.uid() = id);

CREATE POLICY "Users can update own quiz_user record"
ON quiz_users
FOR UPDATE
USING (auth.uid() = id);

CREATE POLICY "Admins can view all quiz_users"
ON quiz_users
FOR SELECT
USING (
  EXISTS (
    SELECT 1 FROM user_roles
    WHERE user_roles.user_id = auth.uid()
    AND user_roles.role = 'admin'
  )
);

-- Cheating logs: insert only, admins can view
CREATE POLICY "Users can insert cheating logs"
ON cheating_logs
FOR INSERT
WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Admins can view cheating logs"
ON cheating_logs
FOR SELECT
USING (
  EXISTS (
    SELECT 1 FROM user_roles
    WHERE user_roles.user_id = auth.uid()
    AND user_roles.role = 'admin'
  )
);
```

**For User Roles:**
```sql
-- Users can read their own role
CREATE POLICY "Users can read own role"
ON user_roles
FOR SELECT
USING (auth.uid() = user_id);

-- Only admins can modify roles
CREATE POLICY "Only admins can modify roles"
ON user_roles
FOR ALL
USING (
  EXISTS (
    SELECT 1 FROM user_roles
    WHERE user_roles.user_id = auth.uid()
    AND user_roles.role = 'admin'
  )
);
```

#### Step 3: Test RLS Policies
```sql
-- Test as non-admin user
SET ROLE authenticated;
SET request.jwt.claim.sub = 'non-admin-user-id';

-- This should return nothing (unless user is admin)
SELECT * FROM questions;

-- Reset
RESET ROLE;
```

---

### 3. Fix Client-Side Authentication Bypass
**Severity:** 🔴 CRITICAL  
**File:** `src/pages/Admin.tsx`

**Problem:**
```typescript
// Only checks localStorage - easily bypassed
const savedAuth = localStorage.getItem('admin_auth');
```

**Solution:**
Replace with Supabase Auth check:

```typescript
import { useAuth } from '@/contexts/AuthContext';
import { useEffect } from 'react';
import { useNavigate } from 'react-router-dom';

const Admin = () => {
  const { user, userRole, loading } = useAuth();
  const navigate = useNavigate();

  useEffect(() => {
    if (!loading) {
      if (!user) {
        navigate('/auth');
      } else if (userRole !== 'admin') {
        navigate('/');
      }
    }
  }, [user, userRole, loading, navigate]);

  if (loading || !user || userRole !== 'admin') {
    return null;
  }

  // Rest of the component...
};
```

**Delete these from Admin.tsx:**
- All useState hooks for authentication
- All useEffect for localStorage checks
- handleLogin and handleLogout functions
- AdminAuth component usage

---

### 4. Improve Session Management
**Severity:** 🔴 CRITICAL  
**Files:** `src/pages/Admin.tsx`, `src/integrations/supabase/client.ts`

**Changes Required:**

1. **Reduce session timeout:**
```typescript
// In Admin.tsx, remove the 24-hour check entirely
// Use Supabase's built-in session management instead
```

2. **Use secure cookies instead of localStorage:**
```typescript
// src/integrations/supabase/client.ts
export const supabase = createClient<Database>(SUPABASE_URL, SUPABASE_PUBLISHABLE_KEY, {
  auth: {
    storage: {
      getItem: (key) => {
        if (typeof document !== 'undefined') {
          return getCookie(key);
        }
        return null;
      },
      setItem: (key, value) => {
        if (typeof document !== 'undefined') {
          setCookie(key, value, {
            secure: true,
            sameSite: 'lax',
            maxAge: 3600 // 1 hour
          });
        }
      },
      removeItem: (key) => {
        if (typeof document !== 'undefined') {
          deleteCookie(key);
        }
      }
    },
    persistSession: true,
    autoRefreshToken: true,
  }
});

// Helper functions for cookies
function getCookie(name: string): string | null {
  const value = `; ${document.cookie}`;
  const parts = value.split(`; ${name}=`);
  if (parts.length === 2) return parts.pop()?.split(';').shift() || null;
  return null;
}

function setCookie(name: string, value: string, options: any) {
  let cookieString = `${name}=${value}`;
  if (options.maxAge) cookieString += `; max-age=${options.maxAge}`;
  if (options.secure) cookieString += '; secure';
  if (options.sameSite) cookieString += `; samesite=${options.sameSite}`;
  cookieString += '; path=/';
  document.cookie = cookieString;
}

function deleteCookie(name: string) {
  document.cookie = `${name}=; max-age=0; path=/`;
}
```

---

### 5. Add Rate Limiting
**Severity:** 🔴 CRITICAL  
**Location:** Authentication endpoints

**Option A: Use Supabase Edge Functions**
Create a Supabase Edge Function with rate limiting:

```typescript
// supabase/functions/login/index.ts
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';

const rateLimiter = new Map<string, { count: number; resetTime: number }>();

serve(async (req) => {
  const ip = req.headers.get('x-forwarded-for') || 'unknown';
  const now = Date.now();
  
  const limit = rateLimiter.get(ip);
  if (limit && limit.resetTime > now && limit.count >= 5) {
    return new Response(
      JSON.stringify({ error: 'Too many login attempts. Try again in 15 minutes.' }),
      { status: 429, headers: { 'Content-Type': 'application/json' } }
    );
  }

  // Reset or increment counter
  if (!limit || limit.resetTime < now) {
    rateLimiter.set(ip, { count: 1, resetTime: now + 900000 }); // 15 minutes
  } else {
    limit.count++;
  }

  // Process login...
  return new Response(JSON.stringify({ success: true }), {
    headers: { 'Content-Type': 'application/json' },
  });
});
```

**Option B: Client-Side Basic Rate Limiting (Temporary)**
```typescript
// src/lib/rateLimit.ts
const attempts = new Map<string, { count: number; resetTime: number }>();

export function checkRateLimit(key: string, maxAttempts: number = 5, windowMs: number = 900000): boolean {
  const now = Date.now();
  const attempt = attempts.get(key);

  if (!attempt || attempt.resetTime < now) {
    attempts.set(key, { count: 1, resetTime: now + windowMs });
    return true;
  }

  if (attempt.count >= maxAttempts) {
    return false;
  }

  attempt.count++;
  return true;
}

export function resetRateLimit(key: string) {
  attempts.delete(key);
}
```

Usage in AdminAuth:
```typescript
import { checkRateLimit, resetRateLimit } from '@/lib/rateLimit';

const handleSubmit = async (e: React.FormEvent) => {
  e.preventDefault();

  if (!checkRateLimit('admin_login')) {
    toast.error('Too many login attempts. Please wait 15 minutes.');
    return;
  }

  try {
    // ... login logic
    resetRateLimit('admin_login'); // Reset on successful login
  } catch (error) {
    // Failed attempt is already counted
  }
};
```

---

### 6. Update Dependencies
**Severity:** 🟠 HIGH  
**Command:**
```bash
npm audit fix
npm update
```

**After running, verify the application still works:**
```bash
npm run build
npm run preview
```

---

### 7. Configure Security Headers
**Severity:** 🟠 HIGH  
**Location:** Deployment platform (Vercel/Netlify) or server configuration

**For Vercel (`vercel.json`):**
```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        },
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "Strict-Transport-Security",
          "value": "max-age=31536000; includeSubDomains"
        },
        {
          "key": "Content-Security-Policy",
          "value": "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self' https://dwzmmclkjlaluxrskvik.supabase.co wss://dwzmmclkjlaluxrskvik.supabase.co"
        },
        {
          "key": "Permissions-Policy",
          "value": "geolocation=(), microphone=(), camera=()"
        },
        {
          "key": "Referrer-Policy",
          "value": "strict-origin-when-cross-origin"
        }
      ]
    }
  ]
}
```

**For Netlify (`netlify.toml`):**
```toml
[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-Content-Type-Options = "nosniff"
    Strict-Transport-Security = "max-age=31536000; includeSubDomains"
    Content-Security-Policy = "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self' https://dwzmmclkjlaluxrskvik.supabase.co wss://dwzmmclkjlaluxrskvik.supabase.co"
    Permissions-Policy = "geolocation=(), microphone=(), camera=()"
    Referrer-Policy = "strict-origin-when-cross-origin"
```

---

## 📝 DEPLOYMENT CHECKLIST

Before deploying to production:

### Phase 1: Critical Security (MUST DO)
- [ ] Remove hardcoded admin credentials (implement proper auth)
- [ ] Enable and test all Supabase RLS policies
- [ ] Configure environment variables in deployment platform
- [ ] Replace localStorage auth with Supabase Auth
- [ ] Test authentication flows thoroughly
- [ ] Verify RLS policies work for both admins and students

### Phase 2: High Priority
- [ ] Run `npm audit fix` and update dependencies
- [ ] Configure security headers in deployment platform
- [ ] Implement rate limiting on authentication
- [ ] Reduce session timeout to 1-2 hours
- [ ] Add CSRF protection where needed

### Phase 3: Testing
- [ ] Test as admin user (all features should work)
- [ ] Test as student user (limited access only)
- [ ] Test as unauthenticated user (no access to protected resources)
- [ ] Try to bypass authentication (should fail)
- [ ] Check browser DevTools for exposed secrets (should find none)
- [ ] Test rate limiting (should block after limit)

### Phase 4: Monitoring
- [ ] Set up error logging (Sentry, LogRocket, etc.)
- [ ] Configure alerts for suspicious activity
- [ ] Monitor authentication failures
- [ ] Review access logs regularly

---

## 🧪 TESTING YOUR FIXES

### Test 1: Credential Exposure
```javascript
// In browser console, search production build
// Should NOT find hardcoded credentials
fetch('/assets/index-[hash].js')
  .then(r => r.text())
  .then(text => {
    console.log('Found admin credentials:', text.includes('iotech2025-26')); // Should be false
    console.log('Found Supabase URL in code:', text.includes('dwzmmclkjlaluxrskvik')); // Should be false (now from env)
  });
```

### Test 2: RLS Policies
```sql
-- In Supabase SQL Editor
-- Try to access data without proper auth
SELECT * FROM questions; -- Should be blocked unless logged in as admin
```

### Test 3: Authentication Bypass
```javascript
// In browser console
localStorage.setItem('admin_auth', JSON.stringify({username: 'hacker', timestamp: Date.now()}));
// Reload page - should NOT grant admin access anymore
```

### Test 4: Rate Limiting
```javascript
// Try multiple failed logins
for (let i = 0; i < 10; i++) {
  // Should be blocked after 5 attempts
}
```

---

## 📞 SUPPORT

If you encounter issues implementing these fixes:

1. **Supabase RLS Issues:** Check Supabase docs at https://supabase.com/docs/guides/auth/row-level-security
2. **Authentication Issues:** Review Supabase Auth docs at https://supabase.com/docs/guides/auth
3. **Build Issues:** Check Vite configuration and environment variable loading

---

## 🎯 ESTIMATED TIME TO COMPLETE

- Phase 1 (Critical): 8-12 hours
- Phase 2 (High Priority): 4-6 hours
- Phase 3 (Testing): 2-4 hours
- **Total: 14-22 hours**

DO NOT deploy to production until at least Phase 1 is complete!

---

*Last Updated: September 13, 2026*
