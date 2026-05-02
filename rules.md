## Workflow Rules

1. **One task at a time** — Complete and confirm before moving to next
2. **Always update `docs/blue-print.md`** after every feature/fix with version number (v1.9.x format)
3. **Read before edit** — Never modify a file without reading it first
4. **No unnecessary changes** — Only touch files directly related to the current task
5. **No code break** — If unsure about safety, ask before changing
6. **Plan before code** — For every task, write a versioned plan `.md` file in `docs/{feature}/` and wait for user approval before writing any code
7. **All images must use `LazyLoadImageCompWithSEO`** — Never use raw `<img>` or `next/image` directly. Always use `@/components/lazyLoadImage/LazyLoadImageCompWithSEO` which provides shimmer skeleton loading + `<noscript>` fallback for crawlers
8. **Plan versioning** — All plans must include a version number in the title (e.g., `v1.10.1`). Use sub-versions for iterations within a plan (v1, v1.1, v2, v2.1). Version must match the `docs/blue-print.md` versioning scheme.
9. **After completion** — Create `{feature}-execution.md` and update `docs/blue-print.md`
10. **All client-side API calls via `fetchClientSide.ts`** — Never call `fetch()` directly in components or hooks. All client-side requests must use the wrapper functions in `lib/fetchClientSide.ts` (which uses `fetchApi` internally). Follow the existing wrapper pattern: typed params, typed `IApiResponse` return, `try/catch` returning `{ success: false, message: Messages.API_ERROR }` on failure.
11. **100% TypeScript — zero errors** — Every new or edited file must be fully typed. No `any` escape hatches to paper over missing types. Run `npx tsc --noEmit` mentally before submitting; if a change would introduce a new TS error, fix it in the same task.

## Architecture

**Next.js 16 + App Router** with MongoDB (Mongoose 9), NextAuth 4 (JWT), Tailwind CSS 4. Uses `@/*` path alias from project root. TypeScript strict mode enabled.

### Multi-Portal Routing

## Security Architecture

### Authentication Flow

- **NextAuth.js** with Credentials provider + JWT strategy (session maxAge: 30 days)
- **Email verification enforced** at login — hard block if `isEmailVerify === false`
- **Session version-based force logout** — incrementing `sessionVersion` invalidates all sessions
- **Password hashing**: bcryptjs with salt rounds 10, minimum 8 characters
- **JWT payload**: user id, roles, isAdmin, status; signed with `NEXTAUTH_SECRET`
- **Access token expiry**: 1 year; reset tokens: 3 days; signup tokens: 30 days

### API Route Protection Pattern

Every `private/` route follows this exact pattern:

```typescript
const authHeader = (await headers()).get("authorization") as string;
const token = authHeader && authHeader.split(" ")[1];
const verification = (await apiJwtGuard(token)) as VerificationResultFromJwt;
if (!verification?.success) return 401;
// Then: isAdmin check for admin routes, ownership filter for client routes
```

### RBAC Enforcement

- **Admin-only**: `if (!verification.isAdmin)` → 400/403
- **Ownership filtering**: `const filter = isAdmin ? {} : { ownersId: userId }`
- **Client routes** (`private/client/*`): verify user is in order's `clients` array

### Input Validation & Sanitization

- **Zod schemas** in `validation/` with `safeParse()` — errors returned as `parsed.error.format()`
- **HTML sanitization**: `validation/sanitizeHTM.ts` uses DOMPurify with strict tag/attribute whitelist
- **Allowed tags**: p, br, strong, em, u, s, h1-h6, blockquote, ul, ol, li, a, pre, code, img, hr, table
- **URI whitelist**: only http, https, mailto, relative URLs
- **External links**: auto-add `rel="noopener noreferrer"`

### Rate Limiting

- Database-driven rate limiter in `lib/security/rateLimit.ts` (no Redis needed)
- Tracks IP (multi-proxy safe: x-forwarded-for, cf-connecting-ip, x-real-ip), user agent, action
- Password reset: 5 attempts per 15 minutes

### File Upload Security

- Route: `app/api/v1/private/upload-aws/`
- **MIME whitelist**: JPEG, PNG, WebP, SVG, Lottie JSON only
- **Size limit**: 10 MB
- **Processing**: sharp resizes to 1400/800/400px, converts to WebP (quality 82)
- **Storage**: S3 with public-read ACL, metadata tracked in Image model
