## prisma usage example
```typescript
//src/actions/profile.action.ts
export async function getProfileByUsername(username: string) {
  try {
    const user = await prisma.user.findUnique({
      where: { username: username },
      select: {
        id: true,
        name: true,
        username: true,
        bio: true,
        image: true,
        location: true,
        website: true,
        createdAt: true,
        _count: {
          select: {
            followers: true,
            following: true,
            posts: true,
          },
        },
      },
    });

    return user;
  } catch (error) {
    console.error("Error fetching profile:", error);
    throw new Error("Failed to fetch profile");
  }
}
//src/app/profile/[username]/page.tsx
type User = Awaited<ReturnType<typeof getProfileByUsername>>;
//typeof will return a fn
//ReturnType will return the return type of that fn which is a promise object
//Awaited will return the actual data after the promise is resolved
```


## Miscellaneous
```typescript
interface ProfilePageClientProps {
  user: NonNullable<User>;
  posts: Posts;
  likedPosts: Posts;
  isFollowing: boolean;
}
//NonNullable is a utility type in TypeScript. Its purpose is to exclude null and undefined from a given type.(not applicable on each property inside it)
```
