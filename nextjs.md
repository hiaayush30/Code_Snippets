# Next.js
**Layout vs. Template Components:**

* `layout.tsx` file persists between switching children pages and does not unmount on these types of changes, allowing for state management.
* Use `template.tsx` if you want components to unmount and mount every time a child page changes.

**Special Route Segments:**

* `[]`: Dynamic route segment.
* `()`: Route segment will be ignored by the routing system.
* `[...]`: Catches everything **after** the previous route segment (e.g., `/blog/[...rest]` will **not** catch `/blog`).
* `[[...]]`: Catches everything, including the route itself (e.g., `/blog/[[...rest]]` will also catch `/blog`).

**Page Configuration:**

* You can export variables other than the default component export in `page.tsx` to tweak Next.js settings.

**Accessing Route Parameters:**

* **Server-side (`params` prop):**
    ```typescript
    ({ params }: { params: { id: string } }) => {
      // Access params.id
    }
    ```
* **Client-side (`useParams` hook):**
    ```typescript
    import { useParams } from 'next/navigation';

    const Page = () => {
      const { id } = useParams();
      // Access id
      return <></>;
    };
    ```

**Special Files:**

* `loading.tsx`: Shows a fallback UI while data is being fetched for a route.
* `error.tsx`: Handles errors that occur during server-side rendering or data fetching within a route.

**Data Revalidation:**

* `router.refresh()`: Fetches API data again without a full page reload (client-side router).

**API Routes (`route.ts`):**

* Can export handler functions for various HTTP methods: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`.
* API routes always run on the server and use Node.js as the default runtime.
* You can change the runtime (e.g., to Edge):
    ```typescript
    export const runtime = 'edge';
    ```

**Next.js and the Edge:**

When we say "Next.js runs on the edge," it doesn't necessarily mean your *entire* Next.js application is duplicated across a massive number of individual, full-fledged servers. Instead, it's more about **deploying specific parts of your application's logic to a distributed network of lightweight compute instances located geographically close to users.**