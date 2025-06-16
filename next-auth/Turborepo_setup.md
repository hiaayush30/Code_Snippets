Excellent! Setting up `next-auth` in a Next.js app within a Turborepo, especially with a shared `db` package and separate API servers (HTTP/WS), is a common and robust pattern.

Here's a comprehensive guide on how to implement `next-auth` in your `nextjs` app, leveraging your existing `db` package and considering the Turborepo structure:

### 1. Prerequisites and Assumptions

* **Turborepo Setup:** You have a working Turborepo with:
    * `apps/nextjs-app` (your Next.js application)
    * `packages/db` (your Prisma client or ORM setup)
    * `apps/http-server` (your Express.js HTTP server) - *Note: NextAuth typically handles its own API routes for authentication, so you might not need to involve your Express HTTP server directly for NextAuth's core auth flow.*
    * `apps/ws-server` (your Express.js WS server) - *Not directly relevant for NextAuth core, but good to note for future WebSocket auth.*
* **`@repo/db/client`:** You've correctly set up and exported `prismaClient` (or your ORM instance) from your `db` package, as seen in your `authOptions` code.
* **TypeScript:** Assumed, given your previous questions.

### 2. Install NextAuth

First, install `next-auth` in your Next.js app:

```bash
cd apps/nextjs-app
pnpm add next-auth
# or yarn add next-auth
# or npm install next-auth
```

### 3. Create/Configure Your `auth` Package (Recommended)

While you *can* put `authOptions` directly in `apps/nextjs-app`, it's a Turborepo best practice to create a dedicated `packages/auth` (or similar) for shared authentication logic, especially if you foresee other apps (like your `http-server`) needing similar authentication or session validation.

**a. Create `packages/auth`:**

```bash
mkdir packages/auth
cd packages/auth
pnpm init # or yarn init or npm init -y
```

**b. Install Dependencies in `packages/auth`:**

```bash
pnpm add next-auth @prisma/client bcryptjs
pnpm add -D prisma typescript
```
*(You might already have `@prisma/client` in `packages/db`, but `next-auth` might list it as a peer dep, and `bcryptjs` is explicitly needed for password hashing.)*

**c. Configure `packages/auth/nextAuth.ts` (Your `authOptions`):**

This is where your `authOptions` (from your previous question) will reside.

**`packages/auth/nextAuth.ts`**

```typescript
import CredentialsProvider from "next-auth/providers/credentials";
import { NextAuthOptions, User as NextAuthUser } from "next-auth";
import { prismaClient } from "@repo/db/client"; // Import from your shared DB package
import bcrypt from "bcryptjs";

// Ensure your Prisma User model has 'id', 'email', 'password' (hashed), 'name', 'photo'
// And that 'id' is a number/Int if not already string.

export const authOptions: NextAuthOptions = {
    providers: [
        CredentialsProvider({
            name: "Credentials",
            credentials: {
                email: { label: "Email", type: "email", placeholder: "email" },
                password: { label: "Password", type: "password", placeholder: "password" }
            },
            async authorize(credentials, _req) { // _req for correct signature
                if (!credentials?.email || !credentials.password) {
                    return null; // Invalid credentials
                }
                try {
                    const user = await prismaClient.user.findFirst({
                        where: {
                            email: credentials.email
                        }
                    });

                    if (!user) {
                        return null; // User not found
                    }

                    const verifyPassword = await bcrypt.compare(credentials.password, user.password);

                    if (!verifyPassword) {
                        return null; // Incorrect password
                    }

                    // Map your Prisma user to NextAuth's User type
                    const authorizedUser: NextAuthUser = {
                        id: user.id.toString(), // CRITICAL: NextAuth expects id as a string
                        email: user.email,
                        name: user.name || null, // Handle potential null/undefined
                        image: user.photo || null, // Map your 'photo' to NextAuth's 'image'
                    };

                    return authorizedUser;

                } catch (error) {
                    console.error('Auth Error', error);
                    return null;
                }
            }
        })
        // Add other providers here if needed (e.g., GoogleProvider, GitHubProvider)
        // For example:
        /*
        GoogleProvider({
            clientId: process.env.GOOGLE_CLIENT_ID!,
            clientSecret: process.env.GOOGLE_CLIENT_SECRET!
        }),
        */
    ],
    session: {
        strategy: "jwt",
        maxAge: 20 * 24 * 60 * 60 // 20 days (NextAuth default is 30 days)
    },
    callbacks: {
        async jwt({ token, user }) { // No need for 'account' and 'profile' here unless using OAuth
            if (user) { // user is only available on first sign-in
                token.id = user.id;
                token.email = user.email;
                token.name = user.name;
                token.picture = user.image; // Map user.image to token.picture
            }
            return token;
        },
        async session({ session, token }) {
            // Ensure these properties are correctly typed in next-auth.d.ts
            if (token.id) {
                session.user.id = token.id;
            }
            if (token.email) {
                session.user.email = token.email;
            }
            if (token.name) {
                session.user.name = token.name;
            }
            if (token.picture) {
                session.user.image = token.picture;
            }
            // If you had other custom fields on the token, add them here
            return session;
        }
    },
    pages: {
        signIn: '/login', // Your custom login page
        error: '/login'   // Redirect errors to login page
    },
    secret: process.env.NEXTAUTH_SECRET, // Must be set in .env
    debug: process.env.NODE_ENV === "development", // Enable debug logs in dev
};
```

**d. Extend NextAuth Types (`packages/auth/next-auth.d.ts`):**

This is crucial for TypeScript to understand your custom `id`, `photo`/`image`, and any other fields you add to the session and JWT token.

**`packages/auth/next-auth.d.ts`**

```typescript
import NextAuth, { DefaultSession } from "next-auth";
import { JWT } from "next-auth/jwt";

declare module "next-auth" {
  /**
   * Returned by `useSession`, `getSession` and received as a prop on the `SessionProvider` React Context
   */
  interface Session {
    user: {
      id: string; // Add id to user
      email: string;
      name: string;
      image: string; // Default NextAuth property for image
      // Add other custom properties you expect on the session user
      // role?: string; // Example for a custom role field
    } & DefaultSession["user"];
  }

  /**
   * The shape of the user object returned in the `authorize` function (via CredentialsProvider)
   * This is NextAuth's internal User type.
   */
  interface User {
    id: string; // Crucial: Must be string for NextAuth
    email: string;
    name?: string | null; // Optional and nullable
    image?: string | null; // Optional and nullable, used for avatar/photo
    // Any other properties returned from authorize must be included here
  }
}

declare module "next-auth/jwt" {
  /**
   * Returned by the `jwt` callback and `getToken`, when using JWT sessions
   */
  interface JWT {
    id: string; // Ensure id is string in JWT
    email: string;
    name?: string | null; // Optional and nullable
    picture?: string | null; // Use 'picture' for the image URL in JWT
    // Any other custom properties you store in the token
    // role?: string; // Example for a custom role field
  }
}
```

**e. Add `packages/auth` to `tsconfig.json` of your `nextjs-app`:**

In `apps/nextjs-app/tsconfig.json`, ensure you have a `paths` mapping for `@repo/auth`:

```json
// apps/nextjs-app/tsconfig.json
{
  "extends": "../../tsconfig.json", // Or your base tsconfig
  "compilerOptions": {
    "plugins": [
      {
        "name": "next"
      }
    ],
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": false,
    "forceConsistentCasingInFileNames": true,
    "noEmit": true,
    "incremental": true,
    "esModuleInterop": true,
    "module": "esnext",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "paths": {
      "@/*": ["./src/*"],
      "@repo/auth/*": ["../../packages/auth/*"], // Add this line
      "@repo/db/*": ["../../packages/db/*"]    // Assuming you already have this
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

### 4. Configure Your Next.js App (`apps/nextjs-app`)

**a. Create `pages/api/auth/[...nextauth].ts` (or `app/api/auth/[...nextauth]/route.ts` for App Router):**

This is NextAuth's catch-all API route.

**For Pages Router (`pages/api/auth/[...nextauth].ts`):**

```typescript
// apps/nextjs-app/pages/api/auth/[...nextauth].ts
import NextAuth from "next-auth";
import { authOptions } from "@repo/auth/nextAuth"; // Import from your auth package

export default NextAuth(authOptions);
```

**For App Router (`app/api/auth/[...nextauth]/route.ts`):**

```typescript
// apps/nextjs-app/app/api/auth/[...nextauth]/route.ts
import NextAuth from "next-auth";
import { authOptions } from "@repo/auth/nextAuth"; // Import from your auth package

const handler = NextAuth(authOptions);

export { handler as GET, handler as POST };
```

**b. Wrap Your App with `SessionProvider`:**

This makes the session available to all components.

**For Pages Router (`pages/_app.tsx`):**

```tsx
// apps/nextjs-app/pages/_app.tsx
import type { AppProps } from 'next/app';
import { SessionProvider } from 'next-auth/react';

function MyApp({ Component, pageProps: { session, ...pageProps } }: AppProps) {
  return (
    <SessionProvider session={session}>
      <Component {...pageProps} />
    </SessionProvider>
  );
}

export default MyApp;
```

**For App Router (`app/layout.tsx`):**

```tsx
// apps/nextjs-app/app/layout.tsx
import { SessionProvider } from './SessionProvider'; // Create this client component
import './globals.css'; // Your global styles

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <SessionProvider>{children}</SessionProvider>
      </body>
    </html>
  );
}

// apps/nextjs-app/app/SessionProvider.tsx (create this file)
// This must be a client component
'use client';

import { SessionProvider } from 'next-auth/react';
import React from 'react';

interface Props {
  children: React.ReactNode;
}

export function SessionProvider({ children }: Props) {
  return <NextAuthSessionProvider>{children}</NextAuthSessionProvider>;
}
```

**c. Create Your Login Page (`pages/login.tsx` or `app/login/page.tsx`):**

This will be your custom login page where users enter credentials.

**For Pages Router (`pages/login.tsx`):**

```tsx
// apps/nextjs-app/pages/login.tsx
import { useState } from 'react';
import { signIn } from 'next-auth/react';
import { useRouter } from 'next/router';

export default function LoginPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const router = useRouter();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError(''); // Clear previous errors

    const result = await signIn('credentials', {
      redirect: false, // Don't redirect automatically
      email,
      password,
      // You can pass the callbackUrl here if you want NextAuth to handle it
      // callbackUrl: router.query.callbackUrl as string || '/dashboard',
    });

    if (result?.error) {
      setError(result.error);
    } else if (result?.ok) {
      // Manually redirect to dashboard or callbackUrl
      router.push((router.query.callbackUrl as string) || '/dashboard');
    }
  };

  return (
    <div>
      <h1>Login</h1>
      <form onSubmit={handleSubmit}>
        {error && <p style={{ color: 'red' }}>{error}</p>}
        <div>
          <label htmlFor="email">Email:</label>
          <input
            type="email"
            id="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            required
          />
        </div>
        <div>
          <label htmlFor="password">Password:</label>
          <input
            type="password"
            id="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
          />
        </div>
        <button type="submit">Sign In</button>
      </form>
      <p>
        Don't have an account? <a href="/signup">Sign Up</a>
      </p>
    </div>
  );
}
```

**For App Router (`app/login/page.tsx`):**

```tsx
// apps/nextjs-app/app/login/page.tsx
'use client'; // This is a client component

import { useState } from 'react';
import { signIn } from 'next-auth/react';
import { useRouter, useSearchParams } from 'next/navigation'; // Use next/navigation for App Router

export default function LoginPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const router = useRouter();
  const searchParams = useSearchParams();
  const callbackUrl = searchParams.get('callbackUrl') || '/dashboard';

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');

    const result = await signIn('credentials', {
      redirect: false,
      email,
      password,
    });

    if (result?.error) {
      setError(result.error);
    } else if (result?.ok) {
      router.push(callbackUrl);
    }
  };

  return (
    <div>
      <h1>Login</h1>
      <form onSubmit={handleSubmit}>
        {error && <p style={{ color: 'red' }}>{error}</p>}
        <div>
          <label htmlFor="email">Email:</label>
          <input
            type="email"
            id="email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            required
          />
        </div>
        <div>
          <label htmlFor="password">Password:</label>
          <input
            type="password"
            id="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
          />
        </div>
        <button type="submit">Sign In</button>
      </form>
      <p>
        Don't have an account? <a href="/signup">Sign Up</a>
      </p>
    </div>
  );
}
```

**d. Implement Signup Page (`pages/signup.tsx` or `app/signup/page.tsx`):**

You'll need a signup page that creates a new user in your database using your `db` package. This typically involves making an API call to your Next.js app (e.g., `/api/register`).

**Example `/api/register` (for Pages or App Router):**

```typescript
// apps/nextjs-app/pages/api/register.ts (or app/api/register/route.ts for App Router)
import type { NextApiRequest, NextApiResponse } from 'next';
import { prismaClient } from '@repo/db/client';
import bcrypt from 'bcryptjs';

export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.method !== 'POST') {
    return res.status(405).json({ message: 'Method Not Allowed' });
  }

  const { email, password, name } = req.body;

  if (!email || !password || !name) {
    return res.status(400).json({ message: 'Email, password, and name are required' });
  }

  try {
    const existingUser = await prismaClient.user.findUnique({
      where: { email },
    });

    if (existingUser) {
      return res.status(409).json({ message: 'User with this email already exists' });
    }

    const hashedPassword = await bcrypt.hash(password, 10); // Hash the password

    const newUser = await prismaClient.user.create({
      data: {
        email,
        password: hashedPassword, // Store the hashed password
        name,
        // photo: '', // Add a default photo if needed
      },
    });

    // You might choose to automatically sign them in after registration
    // Or redirect them to the login page
    return res.status(201).json({ message: 'User registered successfully', user: newUser.email });
  } catch (error) {
    console.error('Registration Error:', error);
    return res.status(500).json({ message: 'Internal server error' });
  }
}
```

Your `signup.tsx` (or `app/signup/page.tsx`) would then call this API route.

**e. Implement Protected Routes/Pages:**

Now, you can use `useSession` to protect components or pages.

```tsx
// apps/nextjs-app/pages/dashboard.tsx (or app/dashboard/page.tsx)
'use client'; // If using App Router

import { useSession, signOut } from 'next-auth/react';
import { useEffect } from 'react';
import { useRouter } from 'next/router'; // For Pages Router
// import { useRouter } from 'next/navigation'; // For App Router

export default function DashboardPage() {
  const { data: session, status } = useSession();
  const router = useRouter();

  useEffect(() => {
    if (status === 'unauthenticated') {
      // Redirect to login if not authenticated
      router.push('/login');
    }
  }, [status, router]);

  if (status === 'loading') {
    return <div>Loading session...</div>;
  }

  if (session) {
    return (
      <div>
        <h1>Welcome to your Dashboard, {session.user?.name || session.user?.email}!</h1>
        <p>Your ID: {session.user?.id}</p>
        {session.user?.image && <img src={session.user.image} alt="User" width={50} height={50} />}
        <button onClick={() => signOut()}>Sign Out</button>
      </div>
    );
  }

  return null; // Should redirect before this is rendered if unauthenticated
}
```

### 5. Environment Variables

Create a `.env.local` file in your `apps/nextjs-app` directory:

```
# apps/nextjs-app/.env.local
NEXTAUTH_SECRET="YOUR_VERY_LONG_RANDOM_STRING_HERE" # Generate with `openssl rand -base64 32` or similar
# If using Google Provider:
# GOOGLE_CLIENT_ID="your_google_client_id"
# GOOGLE_CLIENT_SECRET="your_google_client_secret"
# Add any other provider credentials here
```
**Important:** Your `NEXTAUTH_SECRET` must be strong and kept secret.

### 6. Middleware (`apps/nextjs-app/middleware.ts`)

You've already provided excellent middleware code. Just make sure it points to your `authOptions` correctly:

```typescript
// apps/nextjs-app/middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { authOptions } from '@repo/auth/nextAuth'; // Correct import path
import { getServerSession } from 'next-auth';

// Define protected routes (regex or array)
const protectedRoutes = ['/dashboard', '/settings', '/api/protected'];
const publicRoutes = ['/', '/login', '/signup'];

export default async function middleware(req: NextRequest) {
    const { pathname } = req.nextUrl;

    // getServerSession needs the Request and Response objects in middleware
    // Refer to NextAuth.js middleware documentation for the correct way to get session
    // This typically involves passing req and res (from context) or using auth().
    // For Next.js 13+ App Router, auth() from next-auth/react is used or a custom setup.
    // However, the `getServerSession` directly in middleware as you have it can be problematic
    // due to how `req` and `res` are handled.

    // A more common pattern for Next.js middleware with NextAuth is:
    // 1. Use `auth()` from `next-auth/middleware` (NextAuth v5/Auth.js)
    // 2. Or check `req.cookies` for the session token and manually verify (more complex)
    // 3. Or, for Pages Router or specific setups, pass `req` and `res` to getServerSession

    // Let's assume you're on a setup where getServerSession is intended to work like this
    // If you face issues with session always being null, this is the first place to check.
    const session = await getServerSession(authOptions); // This line is the potential point of failure if not configured right for middleware

    // Check if the current path is a public route
    if (publicRoutes.includes(pathname)) {
        if ((pathname == "/login" || pathname == "/signup") && session?.user) {
            const dashboardUrl = new URL("/dashboard", req.url);
            return NextResponse.redirect(dashboardUrl);
        }
        return NextResponse.next();
    }

    // Authenticate the user

    // If user is not authenticated and trying to access a protected route
    if (!session && protectedRoutes.some(route => pathname.startsWith(route))) {
        const signInUrl = new URL('/login', req.url);
        signInUrl.searchParams.set('callbackUrl', pathname);
        return NextResponse.redirect(signInUrl);
    }

    // If authenticated, allow access
    return NextResponse.next();
}

export const config = {
    matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\.png$).*)'],
};
```
**Important Note on `getServerSession` in Middleware:**
While `getServerSession` *can* be used in middleware for older Next.js versions or specific setups, Next.js's middleware environment is different from API routes. If you're using Next.js 13+ App Router, the recommended approach for authentication in middleware is often to use the `auth` export directly from `next-auth/middleware` (if using NextAuth v5/Auth.js) or a custom session validation based on cookies.

Your current `getServerSession(authOptions)` might work if `authOptions` implicitly handles the request context, but it's a common area of confusion. If `session` is *always* `null` in middleware, that's the first thing to investigate. For most cases, the provided code should work for Pages Router. For App Router, you might need to adapt.

### Summary of Flow:

1.  **User Visits Protected Route:** Middleware intercepts.
2.  **Middleware Checks Session:** Calls `getServerSession`.
3.  **No Session:** Middleware redirects to `/login` (with `callbackUrl`).
4.  **User on `/login`:** Enters credentials.
5.  **`signIn('credentials', ...)` is Called:** Sends request to `/api/auth/callback/credentials`.
6.  **NextAuth API Route Handles:** Calls your `authorize` function in `packages/auth/nextAuth.ts`.
7.  **`authorize` Authenticates:** Verifies credentials against `prismaClient`. Returns a `User` object if successful.
8.  **NextAuth Creates JWT:** Puts user info into the JWT (`jwt` callback).
9.  **NextAuth Sets Session Cookie:** Stores the JWT in an HTTP-only cookie.
10. **NextAuth Redirects:** Redirects to the `callbackUrl` (e.g., `/dashboard`).
11. **User on `/dashboard`:** `useSession` on the client or `getSession` on the server can now read the session from the cookie.

This setup provides a robust and type-safe authentication system in your Next.js application within your Turborepo.
---
## Sharing session with backend app
To extract the **NextAuth token** in your **Next.js (frontend)** app and send it in the `Authorization` header to your **Express backend**, you can follow this setup:

---

### ✅ Step-by-Step Plan

#### 1. **Make sure your NextAuth session returns the token**

In `authOptions`, use the `jwt` and `session` callbacks to include the user `id` or other claims:

```ts
// @repo/auth/nextAuth.ts
import { AuthOptions } from "next-auth";
import CredentialsProvider from "next-auth/providers/credentials";

export const authOptions: AuthOptions = {
  providers: [
    CredentialsProvider({
      // your credentials logic here
    })
  ],
  session: {
    strategy: "jwt",
  },
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.id = user.id;
        token.email = user.email;
      }
      return token;
    },
    async session({ session, token }) {
      if (token) {
        session.user.id = token.id as number;
        session.user.email = token.email as string;
      }
      return session;
    }
  },
  secret: process.env.NEXTAUTH_SECRET,
};
```

> This ensures your JWT has the necessary fields and is accessible on the frontend.

---

#### 2. **Extract token on the client and send to Express**

Use `getToken()` from `next-auth/jwt` **on the backend** (e.g., middleware), or `getSession()` / `getToken()` from `next-auth/react` **on the client**.

But to get the raw JWT on the client, prefer `getToken()` from `next-auth/jwt` via a custom API route or use the session cookie manually.

Instead, here's a clean client-side way using `getSession()`:

```ts
// inside any client-side fetch call
import { getSession } from "next-auth/react";

export async function callExpressAPI() {
  const session = await getSession();

  if (!session) {
    throw new Error("User not authenticated");
  }

  const res = await fetch("http://localhost:4000/protected-route", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${session.accessToken}`, // 👈 OR use your own jwt
    },
  });

  const data = await res.json();
  return data;
}
```

⚠️ **BUT:** `getSession()` does not return the JWT token unless you manually add it in the session callback. You can expose the JWT like this:

```ts
async session({ session, token }) {
  session.accessToken = token.accessToken; // 👈 Add this!
  return session;
}
```

Make sure to type your `Session` object to include `accessToken`:

```ts
declare module "next-auth" {
  interface Session {
    user: {
      id: number;
      email: string;
      photo: string;
      name: string;
    };
    accessToken?: string; // 👈 custom token field
  }
}
```

---

#### 3. **Handle the token in Express middleware**

On your **Express backend**, use a middleware like this:

```ts
// middleware/auth.js
import jwt from "jsonwebtoken";

export function authMiddleware(req, res, next) {
  const authHeader = req.headers["authorization"];
  const token = authHeader?.split(" ")[1];

  if (!token) {
    return res.status(401).json({ error: "Token missing" });
  }

  try {
    const decoded = jwt.verify(token, process.env.NEXTAUTH_SECRET); // 👈 Must match the NextAuth secret
    req.user = decoded;
    next();
  } catch (err) {
    res.status(403).json({ error: "Invalid token" });
  }
}
```

---

### ✅ Summary

| Layer                           | Purpose                                    |
| ------------------------------- | ------------------------------------------ |
| `next-auth`                     | Generates a JWT and includes it in session |
| `getSession()`                  | Extracts session on client                 |
| `session.accessToken`           | Exposes token on frontend                  |
| `Authorization: Bearer <token>` | Sends to Express                           |
| Express middleware              | Verifies JWT using the same secret         |

---

Let me know if you want to avoid JWT and use cookies instead, or want an example with **`next-auth/jwt`** for server-side extraction.
