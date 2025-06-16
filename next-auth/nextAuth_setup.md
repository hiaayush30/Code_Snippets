## Setting up next-auth in Next.js
### Define types for your user in session
- src/types/next-auth.d.ts

```typescript
import "next-auth";
import { DefaultSession } from "next-auth";
declare module "next-auth" {
    // When a user signs in, NextAuth.js creates a Session object that contains information about the user.
    // Represents the user object within the active authentication session. This object contains information about
    // available on Client-side and server-side.
    interface Session {
        user: {
            id: string;
            role: string;
        } & DefaultSession['user']; //name, email, image
        // This means the user object in your Session will now have all the default properties plus your
        // custom id, role properties.
    }
    interface User {
        // Represents the structure of a user record as stored in your database or authentication provider.
        // Primarily server side when interacting with your database to fetch, create, or update user information.
        id:string;
        email: string;
        role: string;
    }
}
```
### Define auth options for next-auth
- src/lib/auth.ts
```typescript
import CredentialsProvider from "next-auth/providers/credentials";
import connectDb from "./db";
import bcrypt from "bcryptjs";
import { NextAuthOptions } from "next-auth";
import User from "@/models/user.model";

export const authOptions: NextAuthOptions = {
    providers: [
        CredentialsProvider({
            name: "Credentials",
            credentials: {
                email: { label: "Email", type: "email", placeholder: "email" },
                password: { label: "Password", type: "password", placeholder: "password" }
            },
            async authorize(credentials) {
                if (!credentials?.email || !credentials.password) {
                    throw new Error('invalid credentials')
                }
                try {
                    await connectDb();
                    const user = await User.findOne({
                        email: credentials.email
                    })
                    if (!user) {
                        throw new Error('user not found')
                    }
                    const verifyPassword = await bcrypt.compare(credentials.password, user.password);
                    if (!verifyPassword) {
                        throw new Error('incorrect password');
                    }
                    return {
                        id: user._id as string,
                        email: user.email,
                        role: user.role
                    }
                } catch (error) {
                    console.error('Auth Error', error);
                    throw error;
                }
            }
        })
    ],
    session: {
        strategy: "jwt",
        maxAge: 20 * 24 * 60 * 60 //30 days
    },
    callbacks: {
        async jwt({ token, user, profile }) {
            // The user object is only available when the user logs in. On subsequent requests, the token is used
            // we are setting what all things will be stored in the token
            if (user) {
                token.id = user.id;
                token.email = user.email;
                token.role = user.role;
            }
            else if (profile) {
                const foundUser = await User.findOne({
                    where: {
                        email: profile.email
                    }
                })
                if (foundUser) {
                    token.id = foundUser.id;
                    token.email = foundUser.email;
                    token.role = foundUser.role;
                }
            }
            return token
        },
        async session({ session, token }) { //extract whatever you need from the token
            session.user.id = token.id as string;
            session.user.role = token.role as string;
            session.user.email = token.email as string;
            return session
        }
    },
    pages: {
        signIn:'/login',
        error: '/login'
    },
    secret: process.env.NEXTAUTH_SECRET
}
```
---
### Create the signup and login pages
#### src/app/(auth)/
- register/page.tsx

[Signup Page](https://github.com/hiaayush30/imagekit-wallpaper-store/blob/main/src/app/(auth)/register/page.tsx)

- login/page.tsx

[Login Page](https://github.com/hiaayush30/imagekit-wallpaper-store/blob/main/src/app/(auth)/login/page.tsx)

---
### Add SessionProvider
```typescript
// app/layout.tsx
import { SessionProvider } from './SessionProvider'; // Custom client component wrapper
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

// app/SessionProvider.tsx (Needs to be a client component for useSession)
'use client'; // This directive marks it as a client component
import { SessionProvider as NextAuthSessionProvider } from 'next-auth/react';
import React from 'react';

interface Props {
  children: React.ReactNode;
}

export default function SessionProvider({ children }: Props) {
  return (
    <NextAuthSessionProvider>
      {children}
    </NextAuthSessionProvider>
  );
}
```
---
### Create the login and register api
- src/app/api/auth/register/route.ts
```typescript
import connectDb from "@/lib/db";
import User from "@/models/user.model";
import { NextRequest, NextResponse } from "next/server";

//if you want various custom info from the user create a register endpoint or the login endpoint will do

export const POST = async (req: NextRequest) => {
    try {
        const { email, password } = await req.json();
        if (!email || !password) {
            return NextResponse.json({
                error: "invalid request,email and password required"
            }, { status: 403 })
        }
        await connectDb();
        const existingUser = await User.findOne({
            email
        });
        if (existingUser) {
            return NextResponse.json({
                error: "User already exists"
            }, { status: 403 })
        }
        await User.create({
            email,
            password,
            role: "user"
        })
        return NextResponse.json({
            message: "User registered successfully"
        }, { status: 201 })
    } catch (error) {
        console.log(error);
        return NextResponse.json({
            error: "Something went wrong!"
        }, { status: 500 })
    }
} 
```
- src/app/api/auth/[...nextauth]/route.ts
```typescript
import { authOptions } from "@/lib/auth";
import NextAuth from "next-auth";


const handler = NextAuth(authOptions);

export { handler as GET, handler as POST }
```
---
### Create Middleware
src/middleware.ts
```typescript
import { withAuth } from "next-auth/middleware"
import { NextResponse } from "next/server"

export default withAuth(
    // `withAuth` augments your `Request` with the user's token.
    function middleware(req) {
        console.log(req.nextauth.token)
        return NextResponse.next();
    },
    {
        callbacks: {
            authorized: ({ token, req }) => {
                const { pathname } = req.nextUrl

                //allow webhook
                if (pathname.startsWith("/api/webhook")) {
                    return true;
                }

                //allow auth related routes
                if (
                    pathname.startsWith("/api/auth") ||
                    pathname.startsWith("/login") ||
                    pathname.startsWith("/register")
                ) {
                    return true;
                }

                //public routes
                if (
                    pathname === "/" ||
                    pathname.startsWith("/api/products") ||
                    pathname.startsWith("/products")
                ) {
                    return true
                }

                //admin routes require admin role
                if (
                    pathname.startsWith("/admin")
                ) {
                    return token?.role === "admin"
                }

                //all other routes require authentication
                return token ? true : false
            },
        },
    },
)

export const config = {
    matcher: [
        // match all routed paths except static files,image optimization files/favicon files and public folder
        "/((?!_next/static|_next/image|favicon.ico|public/).*)"
    ]
}
```
or
```typescript
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { getToken } from 'next-auth/jwt';

export { default } from 'next-auth/middleware';

export async function middleware(request: NextRequest) {
    const token = await getToken({ req: request, secret: process.env.NEXTAUTH_SECRET });
    const url = request.nextUrl; //url that user is requesting for

    // console.log("Token:", token);

    if (token) {
        if(token?.role !== "admin"){
            if(
                url.pathname.startsWith("/admin") ||
                url.pathname.startsWith("/api/admin")
            ){
                return NextResponse.redirect(new URL("/dashboard",request.nextUrl))
            }
        }
        if (
            url.pathname.startsWith('/login') ||
            url.pathname.startsWith('/signup')
        ) {
            return NextResponse.redirect(new URL('/dashboard', request.url));
        }
    } else if (!token) {
        if (url.pathname.startsWith('/dashboard')) {
            return NextResponse.redirect(new URL('/', request.url));
        }
    }

    return NextResponse.next(); // Continue as normal
}

// Apply middleware only to these routes
export const config = {
    matcher: ['/dashboard/:path*', '/login', '/signup','/dashboard'],
};
```

---
### Use it
```typescript
//on client side
await useServerSession();
//on server side
await getServerSession(authOptions);
```