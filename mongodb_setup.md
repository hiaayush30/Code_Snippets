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
            maxPoolSIze: 10
        }
        cached.promise = mongoose.connect(process.env.MONGODB_URI as string, options).then((mongoose) => {
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