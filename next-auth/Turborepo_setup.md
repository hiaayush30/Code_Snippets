Implementing authentication for pages in Next.js within a Turborepo monorepo involves several key considerations, particularly around where to place your authentication logic and how to share it across multiple Next.js applications if you have them.

Here's a breakdown of common strategies and how to apply them in a Turborepo setup:

### Core Concepts of Authentication in Next.js

Before diving into Turborepo specifics, let's review Next.js authentication fundamentals:

1.  **Authentication Provider/Strategy:**
    * **Custom Backend (JWT/Session Cookies):** You handle user registration, login, and token/session management yourself on a custom backend (e.g., Node.js with Express, Python/Django, etc.). Your Next.js app communicates with this backend.
    * **NextAuth.js (Auth.js):** A popular open-source solution that abstracts away much of the complexity. It supports various providers (Google, GitHub, credentials, etc.) and handles sessions, JWTs, and callbacks. Highly recommended for most projects.
    * **Third-Party Auth Services:** Firebase Auth, Auth0, Clerk, Supabase Auth, etc. These are managed services that handle almost all authentication logic for you.

2.  **Session Management:**
    * **Stateless (JWTs):** A JSON Web Token is issued by the server upon successful login and stored on the client (e.g., in `localStorage` or `httpOnly` cookies). The client sends this token with subsequent requests to protected API routes, and the server validates it.
    * **Stateful (Session Cookies):** The server creates a session and stores its ID in a cookie on the client. The server then stores the actual session data (user ID, etc.) in a database or in-memory store.

3.  **Authorization:**
    * **Middleware:** Next.js Middleware (especially with the App Router) runs before a request completes and can be used to protect routes by redirecting unauthenticated users. This is for optimistic checks.
    * **Server Components/Server-Side Rendering (SSR) / `getServerSideProps` (Pages Router):** Fetching user session/token on the server and rendering different UI or redirecting based on authentication status. More secure checks.
    * **Client Components/`useEffect`:** Fetching user session/token on the client and conditionally rendering content or redirecting.

### Authentication in a Turborepo

A Turborepo typically involves multiple `apps/` (like `apps/web`, `apps/admin`) and `packages/` (like `packages/ui`, `packages/auth`).

The key is to decide:
* **Where does your authentication logic live?**
* **How do you share state (user session/token) between Next.js apps or across pages?**

#### Option 1: Using NextAuth.js (Recommended)

NextAuth.js is excellent for monorepos because it provides a unified way to handle authentication.

**Structure in Turborepo:**

1.  **`packages/auth` (or `packages/next-auth-config`):**
    * Create a dedicated package to house your NextAuth.js configuration.
    * This package would export the `authOptions` object (containing providers, callbacks, etc.) that your Next.js apps will use.
    * Example `packages/auth/index.ts`:
        ```typescript
        // packages/auth/index.ts
        import NextAuth from "next-auth";
        import CredentialsProvider from "next-auth/providers/credentials";
        // import GoogleProvider from "next-auth/providers/google";
        // import { verifyCredentials } from './utils'; // Your backend validation function

        export const authOptions = {
            providers: [
                CredentialsProvider({
                    name: "Credentials",
                    credentials: {
                        email: { label: "Email", type: "email" },
                        password: { label: "Password", type: "password" }
                    },
                    async authorize(credentials, req) {
                        // This is where you call your custom backend API to verify credentials
                        // Example:
                        // const res = await fetch("http://localhost:8000/api/user/login", {
                        //     method: "POST",
                        //     headers: { "Content-Type": "application/json" },
                        //     body: JSON.stringify({ email: credentials.email, password: credentials.password }),
                        // });
                        // const user = await res.json();

                        // For demonstration, use dummy user
                        if (credentials.email === "test@example.com" && credentials.password === "password") {
                            return { id: "1", name: "Test User", email: "test@example.com" };
                        }
                        return null; // Return null if user cannot be found/verified
                    },
                }),
                // Add other providers like GoogleProvider, GitHubProvider etc.
            ],
            // Optional: Define pages for signin, error etc.
            pages: {
                signIn: '/auth/signin',
                error: '/auth/error',
            },
            // Optional: Configure session strategy (default is jwt)
            session: {
                strategy: "jwt",
            },
            callbacks: {
                async jwt({ token, user, account, profile }) {
                    // Persist the OAuth access_token and or the user id to the token right after signin
                    if (account) {
                        token.accessToken = account.access_token;
                        token.id = user.id; // Store user ID
                    }
                    return token;
                },
                async session({ session, token }) {
                    // Send properties to the client, like an access_token from a provider.
                    session.accessToken = token.accessToken;
                    session.user.id = token.id; // Expose user ID to client session
                    return session;
                },
            },
            // Add a secret for JWT encryption. Generate a strong one: openssl rand -base64 32
            secret: process.env.NEXTAUTH_SECRET,
            debug: process.env.NODE_ENV === 'development',
        };

        // Export NextAuth handler directly if this package is just for config
        export default NextAuth(authOptions);
        ```

2.  **Next.js App (`apps/web`, `apps/admin`):**
    * **API Route for NextAuth.js:** Each Next.js app needs its own `pages/api/auth/[...nextauth].ts` (Pages Router) or `app/api/auth/[...nextauth]/route.ts` (App Router) to handle NextAuth.js requests.
        ```typescript
        // apps/web/pages/api/auth/[...nextauth].ts (Pages Router)
        // or apps/web/app/api/auth/[...nextauth]/route.ts (App Router)
        import { authOptions } from '@repo/auth'; // Import from your shared package
        import NextAuth from 'next-auth';

        export default NextAuth(authOptions);
        ```
    * **Session Provider:** Wrap your root `_app.js` (Pages Router) or `layout.tsx` (App Router) with `SessionProvider` from `next-auth/react`.
        ```tsx
        // apps/web/pages/_app.tsx (Pages Router)
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

        // apps/web/app/layout.tsx (App Router)
        import { SessionProvider } from './SessionProvider'; // A custom client component wrapper for SessionProvider
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

        // apps/web/app/SessionProvider.tsx (Client component for App Router)
        'use client';
        import { SessionProvider } from 'next-auth/react';
        export default SessionProvider;
        ```
    * **Protecting Pages/Routes:**
        * **Client-Side:** Use `useSession()` hook in React components.
            ```tsx
            import { useSession } from 'next-auth/react';
            import { useRouter } from 'next/navigation'; // or 'next/router' for Pages Router

            function Dashboard() {
              const { data: session, status } = useSession();
              const router = useRouter();

              if (status === 'loading') {
                return <div>Loading...</div>;
              }

              if (status === 'unauthenticated') {
                router.push('/auth/signin');
                return null;
              }

              return (
                <div>
                  <h1>Welcome, {session.user.name}!</h1>
                  <p>You are authenticated.</p>
                </div>
              );
            }
            ```
        * **Server-Side (App Router - Route Handlers / Server Components):** Use `auth()` from `next-auth` (if using NextAuth v5+) or `getServerSession` (older versions) or `cookies().get('next-auth.session-token')`.
            ```typescript
            // apps/web/app/dashboard/page.tsx (Server Component in App Router)
            import { auth } from '@/auth'; // Adjust path based on your setup

            export default async function DashboardPage() {
              const session = await auth(); // Get session on the server

              if (!session) {
                // Redirect unauthenticated users
                // You might need to use `redirect` from 'next/navigation'
                // This would be better handled by Middleware for entire routes
                return <div>Not authenticated. Please sign in.</div>;
              }

              return (
                <div>
                  <h1>Server Welcome, {session.user.name}!</h1>
                  <p>This content is protected.</p>
                </div>
              );
            }
            ```
        * **Middleware (App Router recommended):** Create `middleware.ts` at the root of your Next.js app (e.g., `apps/web/middleware.ts`). This is powerful for protecting entire route groups.
            ```typescript
            // apps/web/middleware.ts
            import { NextResponse } from 'next/server';
            import type { NextRequest } from 'next/server';
            import { auth } from '@repo/auth'; // Import your auth config (NextAuth.js v5+)

            // Define protected routes (regex or array)
            const protectedRoutes = ['/dashboard', '/settings', '/api/protected'];
            const publicRoutes = ['/auth/signin', '/auth/signup', '/'];

            export default async function middleware(req: NextRequest) {
              const { pathname } = req.nextUrl;

              // Check if the current path is a public route
              if (publicRoutes.includes(pathname) || pathname.startsWith('/_next')) {
                return NextResponse.next();
              }

              // Authenticate the user
              const session = await auth(); // Using auth() from NextAuth.js v5+

              // If user is not authenticated and trying to access a protected route
              if (!session && protectedRoutes.some(route => pathname.startsWith(route))) {
                const signInUrl = new URL('/auth/signin', req.url);
                signInUrl.searchParams.set('callbackUrl', pathname); // Optional: redirect back after login
                return NextResponse.redirect(signInUrl);
              }

              // If authenticated, allow access
              return NextResponse.next();
            }

            // Define which paths the middleware should run on
            export const config = {
              // Match all routes except static files, _next, and public images
              matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\.png$).*)'],
            };
            ```

#### Option 2: Custom JWT/Session Implementation

If you prefer a custom solution (e.g., you already have a backend authentication service):

1.  **Backend:** Your backend handles user login, generates JWTs, and sets them in `httpOnly` cookies (recommended for security) or sends them in the response body for `localStorage` storage.
2.  **Shared `packages/auth-utils`:**
    * Create a package like `packages/auth-utils` that contains utility functions for:
        * Storing/retrieving tokens from `localStorage` or cookies.
        * Helper functions for making authenticated API requests.
        * Context providers (React Context API) to manage authentication state across your Next.js apps.
3.  **Next.js App (`apps/web`, `apps/admin`):**
    * **Login Page:** On your login page, you'd send credentials to your backend.
    * **Store Token:** Upon successful login, store the JWT in `localStorage` or a cookie.
    * **Protecting Pages:**
        * **Client-Side:** In `_app.js` or `layout.tsx`, or in individual components, check for the presence of the token. If absent, redirect.
            ```typescript
            // Client-side component example
            import { useEffect, useState } from 'react';
            import { useRouter } from 'next/navigation'; // or 'next/router'

            function ProtectedPage() {
              const [isLoading, setIsLoading] = useState(true);
              const router = useRouter();

              useEffect(() => {
                const token = localStorage.getItem('authToken'); // Or read from cookie
                if (!token) {
                  router.push('/login'); // Redirect to login
                } else {
                  // Optionally, validate token with backend or decode it
                  setIsLoading(false);
                }
              }, [router]);

              if (isLoading) {
                return <div>Loading authentication...</div>;
              }

              return (
                <div>
                  <h1>Welcome to the protected area!</h1>
                </div>
              );
            }
            ```
        * **Server-Side (`getServerSideProps` for Pages Router, Middleware/Server Components for App Router):** This is crucial for truly protected routes and better SEO.
            ```typescript
            // Pages Router example: pages/protected.tsx
            import { GetServerSideProps } from 'next';

            export const getServerSideProps: GetServerSideProps = async (context) => {
              const token = context.req.cookies.authToken; // Get token from server-side request cookies

              if (!token) {
                return {
                  redirect: {
                    destination: '/login',
                    permanent: false,
                  },
                };
              }

              // Optionally: Validate token with your backend API
              // const res = await fetch('http://your-backend.com/api/validate-token', {
              //   headers: { Authorization: `Bearer ${token}` }
              // });
              // if (!res.ok) {
              //   return { redirect: { destination: '/login', permanent: false } };
              // }

              return {
                props: {}, // Pass data to the page if needed
              };
            };

            function ProtectedPage() {
              return <div>This is a server-protected page.</div>;
            }
            export default ProtectedPage;
            ```
            For the App Router, you'd use Middleware and checks within Server Components.

### Key Turborepo Benefits for Auth:

* **Shared Code:** By placing common authentication logic (like NextAuth.js config, or custom auth utilities) in a `packages/` directory, you avoid duplicating code across your Next.js apps.
* **Centralized Configuration:** All apps consume the same authentication rules and providers.
* **Easier Updates:** Update your `packages/auth` package, and all apps that depend on it get the updates.
* **Monorepo Benefits:** Faster builds and caching with Turborepo for your auth package.

### Summary Steps:

1.  **Choose an Auth Strategy:** NextAuth.js is generally recommended for its features and community support.
2.  **Create Shared Auth Package:** Make a `packages/auth` (or similar) to house your common authentication logic and configurations.
3.  **Integrate into Next.js Apps:**
    * Set up API routes for authentication.
    * Wrap your apps with a session provider.
    * Implement client-side and server-side (using Middleware or `getServerSideProps`/Server Components) protection.
4.  **Environment Variables:** Remember to manage environment variables (like `NEXTAUTH_SECRET`, API keys, backend URLs) in your Turborepo's root `.env` file or within each app's `.env` if they differ, and ensure they are exposed correctly.

Implementing authentication is a critical part of any application, so take your time to understand the chosen method and its security implications.