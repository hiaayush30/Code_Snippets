## Setting up Mongodb in Next.js

- types.d.ts
```typescript
import { Connection } from "mongoose";

declare global {
    // eslint-disable-next-line no-var
    var mongoose: { 
      conn: Connection | null;
      promise: Promise<Connection> | null;
    };
  }
```

- lib/db.ts

```typescript
import mongoose from "mongoose";

// Global variable to store the connection
let cached = global.mongoose;

if (!cached) {
    cached = global.mongoose = { conn: null, promise: null };
}

async function connectDb() {
    if (cached.conn) {
        return cached.conn;
    }

    if (!cached.promise) {
        const options = {
            bufferCommands: true,
            // tells Mongoose to queue up operations on your models if the connection to 
            // MongoDB hasn't been established yet.
            maxPoolSIze: 10
        }
        cached.promise = mongoose.connect(process.env.MONGODB_URI as string, options)
            .then((mongoose) => {
            return mongoose.connection;
        });
    }

    try {
        cached.conn = await cached.promise;
    } catch (e) {
        cached.promise = null;
        throw e;
    }

    return cached.conn;
}

export default connectDb;
```    

- models/user.model.ts

```typescript
const User:Model<IUser> = mongoose.models['User'] ? mongoose.models['User'] : mongoose.model<IUser>('User', userSchema);
```

. using it
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