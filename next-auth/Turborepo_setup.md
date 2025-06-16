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
It's a great question! Sharing session state between your Next.js frontend (powered by NextAuth.js) and your separate Express.js HTTP server backend in a Turborepo is a common requirement for full-stack applications.

NextAuth.js primarily uses **JSON Web Tokens (JWTs)** for its sessions by default (when `session.strategy` is set to `"jwt"`). This is excellent for cross-service authentication, as JWTs are stateless and can be validated independently.

Here's how you can leverage the NextAuth.js session in your Express.js HTTP server's middleware:

### 1. Understanding NextAuth.js JWTs

When a user successfully logs in via NextAuth.js, a JWT is created and stored in an HTTP-only cookie in the user's browser. This cookie is automatically sent with every subsequent request from the browser to your Next.js application.

The JWT contains the `token` object that you configured in your `jwt` callback (e.g., `id`, `email`, `name`, `picture`). This token is cryptographically signed using your `NEXTAUTH_SECRET`.

### 2. Key Challenges for the Express Server

* **Cookie Accessibility:** Your Express server *won't* automatically receive the NextAuth.js session cookie **if your Express server is on a different domain or port** than your Next.js app. This is a fundamental browser security restriction (Same-Origin Policy).
* **JWT Verification:** Your Express server needs to be able to decode and verify the JWT using the *same `NEXTAUTH_SECRET`* that NextAuth.js uses.

### 3. Solution Strategy: Pass JWT as a Bearer Token

The most robust and secure way to share the session is to **extract the JWT from the cookie on the Next.js frontend and pass it as an `Authorization: Bearer <token>` header** to your Express.js API requests.

This approach works regardless of whether your Next.js and Express apps are on the same domain/port, and it's a standard pattern for API authentication.

---

### Step-by-Step Implementation

#### A. Configure `next-auth` to retrieve the JWT (Frontend - `apps/nextjs-app`)

You need a way to get the JWT client-side. The `getToken` utility from `next-auth/jwt` is perfect for this.

**Create a utility function or use it directly:**

```typescript
// apps/nextjs-app/utils/auth.ts (or directly in your component/API client)
import { getToken } from 'next-auth/jwt';
import { NextApiRequest } from 'next'; // Only if using in Next.js API routes

const secret = process.env.NEXTAUTH_SECRET;

export async function getJwtToken(req?: NextApiRequest) {
  if (!secret) {
    throw new Error("NEXTAUTH_SECRET is not defined");
  }
  // If `req` is provided (e.g., in a Next.js API route), getToken can read from headers/cookies
  // Otherwise, it will try to read from the browser's context
  const token = await getToken({ req: req || undefined, secret });
  return token;
}
```

#### B. Modify API Calls from Next.js to Express (Frontend - `apps/nextjs-app`)

When your Next.js app makes requests to your Express.js HTTP server (e.g., `axios.get('http://localhost:8000/api/mydata')`), you need to include the JWT.

```typescript
// apps/nextjs-app/components/MyProtectedComponent.tsx (Client Component)
'use client'; // if using App Router

import { useState, useEffect } from 'react';
import axios from 'axios';
import { useSession } from 'next-auth/react'; // To ensure user is logged in
import { getJwtToken } from '../utils/auth'; // Your utility to get the token

export default function MyProtectedComponent() {
  const { data: session, status } = useSession();
  const [data, setData] = useState(null);
  const [error, setError] = useState('');

  useEffect(() => {
    const fetchData = async () => {
      if (status === 'authenticated') {
        try {
          const token = await getJwtToken(); // Get the JWT token
          if (!token) {
            setError('No authentication token found.');
            return;
          }

          const response = await axios.get('http://localhost:8000/api/protected-data', {
            headers: {
              Authorization: `Bearer ${token.raw}`, // Use token.raw which is the actual JWT string
            },
          });
          setData(response.data);
        } catch (err: any) {
          setError('Failed to fetch data: ' + err.message);
        }
      }
    };

    fetchData();
  }, [status]); // Re-fetch if auth status changes

  if (status === 'loading') return <div>Loading...</div>;
  if (status === 'unauthenticated') return <div>Please log in to view this content.</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div>
      <h2>Protected Data from Express</h2>
      {data ? <pre>{JSON.stringify(data, null, 2)}</pre> : <p>Loading data...</p>}
    </div>
  );
}
```

#### C. Create Authentication Middleware in Express (`apps/http-server`)

Your Express server needs to parse the `Authorization` header, verify the JWT, and attach the decoded user information to the `req` object.

**1. Install Dependencies in `apps/http-server`:**

```bash
cd apps/http-server
pnpm add jsonwebtoken dotenv
pnpm add -D @types/jsonwebtoken @types/express
```
* `jsonwebtoken`: For verifying JWTs.
* `dotenv`: To load your `NEXTAUTH_SECRET` from `.env`.

**2. Create the Authentication Middleware:**

```typescript
// apps/http-server/src/middleware/auth.ts (create this file)
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import dotenv from 'dotenv';

dotenv.config(); // Load environment variables from .env

const JWT_SECRET = process.env.NEXTAUTH_SECRET;

// Extend the Request type to include user information
declare global {
  namespace Express {
    interface Request {
      user?: {
        id: string;
        email: string;
        name?: string | null;
        image?: string | null;
        // Add any other properties you store in your NextAuth JWT token
      };
    }
  }
}

export const authenticateToken = (req: Request, res: Response, next: NextFunction) => {
  const authHeader = req.headers['authorization'];
  const token = authHeader && authHeader.split(' ')[1]; // Bearer TOKEN

  if (token == null) {
    return res.sendStatus(401); // No token provided
  }

  if (!JWT_SECRET) {
    console.error("JWT_SECRET is not defined in http-server environment.");
    return res.status(500).send("Server configuration error.");
  }

  jwt.verify(token, JWT_SECRET, (err, user) => {
    if (err) {
      console.error("JWT verification failed:", err.message);
      return res.sendStatus(403); // Token is invalid or expired
    }
    // 'user' here is the decoded payload of the JWT.
    // Ensure it matches the structure you put in the NextAuth JWT callback.
    // In NextAuth's `jwt` callback, `token` has `id`, `email`, `name`, `picture`.
    // So, 'user' here will have those same properties.
    req.user = user as Request['user']; // Attach the decoded user to the request
    next(); // Proceed to the next middleware/route handler
  });
};
```

#### D. Use the Middleware in Your Express App (`apps/http-server`)

```typescript
// apps/http-server/src/index.ts (or wherever your Express app is configured)
import express from 'express';
import cors from 'cors'; // Important for cross-origin requests
import { authenticateToken } from './middleware/auth'; // Import your auth middleware

const app = express();
const port = process.env.PORT || 8000;

app.use(express.json()); // For parsing JSON request bodies

// Configure CORS
app.use(cors({
  origin: 'http://localhost:3000', // Allow requests from your Next.js app's origin
  credentials: true, // Allow cookies/authorization headers to be sent
}));

app.get('/', (req, res) => {
  res.send('Hello from Express HTTP Server!');
});

// Example protected route
app.get('/api/protected-data', authenticateToken, (req, res) => {
  // If we reach here, the token was valid, and req.user is populated
  if (req.user) {
    res.json({
      message: 'This is protected data!',
      userId: req.user.id,
      userEmail: req.user.email,
      userName: req.user.name,
      // You can return sensitive data if appropriate, as the user is authenticated
    });
  } else {
    // This case theoretically shouldn't be reached if authenticateToken works
    res.status(500).json({ message: 'User not found in request context.' });
  }
});

app.listen(port, () => {
  console.log(`HTTP server listening on port ${port}`);
});
```

#### E. Environment Variables for Express Server (`apps/http-server`)

Create a `.env` file in your `apps/http-server` directory (or ensure it's loaded from your Turborepo root if you have a shared `.env` strategy).

```
# apps/http-server/.env
PORT=8000
NEXTAUTH_SECRET="YOUR_VERY_LONG_RANDOM_STRING_HERE" # MUST BE THE SAME AS YOUR NEXT.JS APP'S SECRET
```

### Key Considerations:

1.  **`NEXTAUTH_SECRET` Consistency:** The `NEXTAUTH_SECRET` environment variable **must be identical** in both your Next.js application (`apps/nextjs-app/.env.local`) and your Express.js HTTP server (`apps/http-server/.env`). This is critical for JWT signing and verification.
2.  **CORS:** Cross-Origin Resource Sharing (`cors` package in Express) is essential if your Next.js app and Express server run on different origins (different ports or domains). Configure it carefully to allow requests from your Next.js app's URL.
3.  **`token.raw`:** When you get the token using `getToken` on the client, `token.raw` holds the actual JWT string that you need to send in the `Authorization` header.
4.  **Error Handling:** Ensure robust error handling for failed token verification (e.g., returning 401 Unauthorized or 403 Forbidden).
5.  **Typescript `Request` Augmentation:** The `declare global` block is necessary to tell TypeScript that your `Request` object will have a `user` property after the `authenticateToken` middleware runs.
6.  **Refresh Tokens (Advanced):** For long-lived sessions, you might eventually look into refresh tokens if you want to avoid extremely long JWT expiration times, but for basic usage, NextAuth's default JWT strategy with reasonable `maxAge` is usually sufficient.
7.  **WebSockets (`ws-server`):** For your WS server, you would typically pass the JWT token during the WebSocket connection handshake (e.g., as a query parameter or header when establishing the WS connection) and then verify it on the server side using `jsonwebtoken` similarly.

This setup provides a secure and scalable way to authenticate requests to your Express.js backend using the NextAuth.js session.
You've hit on a crucial point when integrating Next.js with separate Express.js servers in a monorepo: **sharing authentication state**. NextAuth.js primarily manages sessions for the Next.js application itself. To use that session information in your Express.js HTTP server, you need to either:

1.  **Verify the NextAuth.js JWT on your Express.js server.** (Recommended and most common)
2.  **Use a shared session store (like Redis) and make Express.js understand it.** (More complex)
3.  **Use `@auth/express` (experimental).**

Let's focus on **Option 1: Verifying the JWT**. This is the standard approach for stateless API servers.

### Understanding NextAuth.js JWTs

When you configure NextAuth.js with `session: { strategy: "jwt" }`, it doesn't store session data in a database by default. Instead, it creates a JSON Web Token (JWT) after a successful login. This JWT is then signed with your `NEXTAUTH_SECRET` and sent as an `HttpOnly` cookie to the client's browser.

Crucially, this JWT contains the `token` object from your `jwt` callback in `authOptions`, which includes `user.id`, `user.email`, `user.name`, `user.picture`, and any other custom data you added.

### Steps to Use NextAuth.js Session in Your Express.js HTTP Server

**Goal:** Your Express.js middleware needs to:
1.  Read the `next-auth.session-token` cookie from incoming requests.
2.  Verify the JWT using the same `NEXTAUTH_SECRET` that NextAuth.js uses.
3.  Decode the JWT to extract the user's information.
4.  Attach this user information to the `req` object for subsequent route handlers to use.

---

### **1. Shared Secret Key (Critical)**

Both your Next.js app (via NextAuth.js) and your Express.js server **MUST** use the exact same `NEXTAUTH_SECRET` for signing and verifying JWTs.

**In your Turborepo:**
* Make sure `NEXTAUTH_SECRET` is set in the `.env` file of your `apps/nextjs-app`.
* **Also set `NEXTAUTH_SECRET` in the `.env` file (or equivalent configuration) of your `apps/http-server`.**

### **2. Install Dependencies in `apps/http-server`**

You'll need a JWT library to verify the token.
`jsonwebtoken` is a popular choice.

```bash
cd apps/http-server
pnpm add jsonwebtoken dotenv
pnpm add -D @types/jsonwebtoken # For TypeScript
```
* `dotenv` is for loading environment variables.

### **3. Create an Authentication Middleware in `apps/http-server`**

You'll create a middleware function that intercepts requests, extracts the JWT, verifies it, and attaches the user data.

**`apps/http-server/src/middleware/auth.ts`**

```typescript
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { JWT_VERIFICATION_ERROR } from '../utils/constants'; // Define this constant for clear errors

// Extend the Request object to include the user property
declare global {
    namespace Express {
        interface Request {
            user?: {
                id: string;
                email: string;
                name?: string | null;
                image?: string | null;
                // Add any other properties you added to your JWT token in NextAuth.js
                // role?: string;
            };
        }
    }
}

interface DecodedToken {
    id: string;
    email: string;
    name?: string | null;
    picture?: string | null; // NextAuth uses 'picture' in the JWT
    iat: number;
    exp: number;
    // Add any other custom properties from your JWT
    // role?: string;
}

export const authenticateToken = (req: Request, res: Response, next: NextFunction) => {
    // 1. Get the session token from the cookies
    // NextAuth.js uses either 'next-auth.session-token' or '__Secure-next-auth.session-token'
    // depending on the environment (HTTPS vs HTTP) and configuration.
    const token = req.cookies['next-auth.session-token'] || req.cookies['__Secure-next-auth.session-token'];

    if (!token) {
        return res.status(401).json({ message: 'Authentication required: No token provided' });
    }

    try {
        // 2. Verify and decode the JWT
        // Use the same NEXTAUTH_SECRET as in your Next.js app
        const secret = process.env.NEXTAUTH_SECRET;

        if (!secret) {
            console.error('NEXTAUTH_SECRET is not defined in http-server environment.');
            return res.status(500).json({ message: 'Server configuration error: Authentication secret missing.' });
        }

        const decoded = jwt.verify(token, secret) as DecodedToken;

        // 3. Attach user information to the request object
        req.user = {
            id: decoded.id,
            email: decoded.email,
            name: decoded.name,
            image: decoded.picture, // Map 'picture' from JWT back to 'image' for consistency
            // role: decoded.role, // If you added a role
        };

        // 4. Proceed to the next middleware or route handler
        next();

    } catch (error) {
        console.error('JWT Verification Error:', error);
        // Handle token expiration, invalid token, etc.
        if (error instanceof jwt.TokenExpiredError) {
            return res.status(401).json({ message: 'Authentication required: Token expired' });
        }
        return res.status(401).json({ message: 'Authentication failed: Invalid token' });
    }
};

// You might also want a separate file for constants, e.g., `apps/http-server/src/utils/constants.ts`
// export const JWT_VERIFICATION_ERROR = 'Authentication failed: Invalid token';
```

### **4. Integrate the Middleware into Your Express.js App**

In your main Express.js application file (`apps/http-server/src/index.ts` or similar):

```typescript
// apps/http-server/src/index.ts (or your main app file)
import express from 'express';
import dotenv from 'dotenv';
import cookieParser from 'cookie-parser'; // Important: to parse cookies from the request
import { authenticateToken } from './middleware/auth'; // Adjust path as needed

// Load environment variables
dotenv.config();

const app = express();
const PORT = process.env.PORT || 8000;

// Middleware setup
app.use(express.json()); // For parsing JSON request bodies
app.use(cookieParser()); // Essential for accessing req.cookies

// Example protected route
app.get('/api/protected-data', authenticateToken, (req, res) => {
    // If we reach here, the user is authenticated, and req.user is populated
    if (req.user) {
        res.json({
            message: 'This is protected data!',
            user: req.user,
            data: { secretValue: 'You are authenticated!' }
        });
    } else {
        // This case should ideally not happen if authenticateToken works correctly
        res.status(500).json({ message: 'User information missing after authentication.' });
    }
});

// Example unprotected route
app.get('/api/public-data', (req, res) => {
    res.json({ message: 'This is public data!' });
});

app.listen(PORT, () => {
    console.log(`HTTP Server running on port ${PORT}`);
});
```

### **5. Client-Side Interaction from Next.js App**

When your Next.js frontend makes requests to your Express.js backend, the browser will automatically send the `next-auth.session-token` cookie with the request (because it's an HttpOnly cookie set by NextAuth.js on the same domain or a subdomain configured to share cookies).

You don't need to manually attach headers or tokens in your `fetch` or `axios` calls from the Next.js app if they are on the same domain/subdomain.

```typescript
// apps/nextjs-app/src/components/SomeComponent.tsx
'use client';

import { useSession } from 'next-auth/react';
import { useEffect, useState } from 'react';

export default function SomeComponent() {
  const { data: session } = useSession();
  const [protectedData, setProtectedData] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    const fetchProtectedData = async () => {
      if (session) {
        try {
          // Assuming your Express server is on a different port but same base domain
          // Or a subdomain configured to share cookies
          // e.g., Next.js on localhost:3000, Express on localhost:8000
          const response = await fetch('http://localhost:8000/api/protected-data');

          if (!response.ok) {
            // Handle 401, 403 etc.
            const errData = await response.json();
            throw new Error(errData.message || 'Failed to fetch protected data');
          }

          const data = await response.json();
          setProtectedData(data);
        } catch (err: any) {
          setError(err.message);
        }
      }
    };

    fetchProtectedData();
  }, [session]); // Re-run when session changes

  if (!session) {
    return <div>Please log in to see protected data.</div>;
  }

  if (error) {
    return <div>Error fetching data: {error}</div>;
  }

  return (
    <div>
      <h2>Protected Data from Express Server:</h2>
      {protectedData ? (
        <pre>{JSON.stringify(protectedData, null, 2)}</pre>
      ) : (
        <div>Loading protected data...</div>
      )}
    </div>
  );
}
```

### Key Considerations for Turborepo and Monorepo:

1.  **Environment Variables:** Ensure `NEXTAUTH_SECRET` is consistently available to both your Next.js app and your Express.js app. Using a shared `.env` or a robust env management system for your Turborepo is crucial.
2.  **CORS:** If your Next.js app and Express.js server are running on *different origins* (e.g., `localhost:3000` and `localhost:8000`), you will encounter Cross-Origin Resource Sharing (CORS) issues.
    * You'll need to configure your Express.js server to allow CORS requests from your Next.js app's origin.
    * **Crucially for cookies:** You must set `credentials: 'include'` in your client-side `fetch` or `axios` calls and ensure your CORS configuration on the Express.js server explicitly allows credentials (`Access-Control-Allow-Credentials: true`).

    ```typescript
    // apps/http-server/src/index.ts (CORS example)
    import cors from 'cors'; // npm install cors

    // ...
    app.use(cookieParser());

    app.use(cors({
        origin: 'http://localhost:3000', // Your Next.js app's origin
        credentials: true, // Allow cookies to be sent
    }));
    // ...
    ```

3.  **Domain/Subdomain Consistency:** For `HttpOnly` cookies to be sent automatically across different ports, they generally need to be on the same base domain or a common subdomain.
    * If running locally, `localhost` typically works across different ports.
    * In production, if `nextjs.yourdomain.com` and `api.yourdomain.com`, you'd need to configure NextAuth's cookie domain (usually done by setting `NEXTAUTH_URL` correctly) and ensure your Express.js server is listening on the appropriate subdomain.

By following these steps, you can effectively use the authentication state managed by NextAuth.js in your Next.js application to secure routes and access user information in your separate Express.js HTTP server.