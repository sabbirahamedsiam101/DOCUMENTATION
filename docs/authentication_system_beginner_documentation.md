# Authentication System with Access Token, Refresh Token & Sessions

> A beginner-friendly guide to understanding and practicing a modern
> authentication system with Node.js, Express, MongoDB, React, and RTK
> Query.

------------------------------------------------------------------------

## 1. What You Will Learn

After completing this documentation, you should understand:

-   What an access token is
-   What a refresh token is
-   Why access tokens are short-lived
-   Why refresh tokens are long-lived
-   When the frontend should refresh an access token
-   Who triggers the refresh request
-   Refresh-token rotation
-   Why refresh tokens should be hashed in the database
-   What a session record represents
-   How logout works
-   What happens to old/revoked refresh tokens
-   How frontend and backend work together
-   How to build a simple authentication practice project

------------------------------------------------------------------------

# Part 1 --- The Big Picture

## 2. Access Token vs Refresh Token

Authentication commonly uses two different credentials.

### Access Token

The access token answers:

> "Can this request access a protected resource right now?"

It should normally have a short lifetime.

Example:

``` text
Access Token Lifetime = 15 minutes
```

The client sends it with protected API requests:

``` http
Authorization: Bearer ACCESS_TOKEN
```

Example:

``` text
GET /api/users/me
        |
        v
Authorization: Bearer eyJ...
```

The server verifies the token.

If it is valid, the request continues.

If it is expired or invalid:

``` text
401 Unauthorized
```

------------------------------------------------------------------------

## 3. Why Is the Access Token Short-Lived?

Imagine an attacker somehow obtains an access token.

If it lives for 30 days, the attacker may have access for 30 days.

If it lives for 15 minutes, the attacker's usable window is much
smaller.

So the basic idea is:

``` text
Access Token
     |
     v
Short lifetime
     |
     v
Reduced damage if stolen
```

A short-lived access token is not a complete security solution, but it
limits the lifetime of a stolen access credential.

------------------------------------------------------------------------

# 4. Refresh Token

The refresh token answers:

> "Is this client still allowed to obtain a new access token?"

Example:

``` text
Refresh Token Lifetime = 7 days
```

The refresh token is not normally sent with every API request.

Instead, it is used with a dedicated endpoint:

``` http
POST /api/auth/refresh
```

The server validates the refresh token and issues a new access token.

------------------------------------------------------------------------

# 5. Why Do We Need Both?

Without refresh tokens, you have two bad choices.

### Option 1 --- Very short access token

``` text
Access token = 15 minutes
```

The user would have to log in again every 15 minutes.

Terrible user experience.

### Option 2 --- Very long access token

``` text
Access token = 30 days
```

A stolen access token remains useful for a long time.

Also undesirable.

### Better approach

``` text
Access Token
    15 minutes
        |
        v
expires
        |
        v
Refresh Token
    7 days
        |
        v
new Access Token
```

The user stays logged in without making the access token itself
long-lived.

------------------------------------------------------------------------

# Part 2 --- The Authentication Lifecycle

## 6. Login

Suppose a user logs in at 10:00 AM.

``` http
POST /api/auth/login
```

The server:

1.  Finds the user.
2.  Verifies the password.
3.  Creates an access token.
4.  Creates a refresh token.
5.  Hashes the refresh token.
6.  Creates a session record.
7.  Sends the access token to the client.
8.  Stores the refresh token in a secure HttpOnly cookie.

Example:

``` text
10:00 AM

Access Token
expires at 10:15 AM

Refresh Token
expires after 7 days
```

------------------------------------------------------------------------

# 7. Where Should the Tokens Be Stored?

A common architecture for a browser application is:

``` text
Browser
|
|-- Access Token
|      |
|      +-- JavaScript memory/state
|
+-- Refresh Token
       |
       +-- HttpOnly Cookie
```

## Access Token

The frontend can keep the access token in memory or application state.

For example:

``` js
let accessToken = null;
```

In a real React application, your authentication/API layer manages this
more systematically.

## Refresh Token

The refresh token should ideally be stored in an:

``` text
HttpOnly
Secure
SameSite
```

cookie.

The important part is `HttpOnly`.

JavaScript cannot directly read an HttpOnly cookie.

That reduces exposure to token theft through client-side JavaScript.

------------------------------------------------------------------------

# 8. Session Records

A refresh token represents a long-lived authentication session.

Instead of storing the actual refresh token in MongoDB, store a hash of
it.

A simple session model for this practice project is:

``` js
{
  userId,
  refreshTokenHash,
  ip,
  userAgent,
  revoke
}
```

with:

``` js
{ timestamps: true }
```

So MongoDB automatically gives us:

``` text
createdAt
updatedAt
```

Example document:

``` js
{
  _id: "...",
  userId: "...",
  refreshTokenHash: "...",
  ip: "192.168.1.10",
  userAgent: "Chrome on Windows",
  revoke: false,
  createdAt: "...",
  updatedAt: "..."
}
```

------------------------------------------------------------------------

# 9. Why Store a Hash Instead of the Refresh Token?

Never unnecessarily store a raw long-lived credential.

Suppose your database contains:

``` text
refreshToken:
abc123-secret-token
```

If somebody obtains a database dump, they may be able to use that token.

Instead:

``` text
Refresh Token
      |
      v
   Hashing
      |
      v
Refresh Token Hash
      |
      v
MongoDB
```

The browser has:

``` text
Refresh Token = abc123...
```

The database has:

``` text
Hash = 9f4c...
```

When the refresh request arrives, the server hashes the presented token
and compares it with the stored hash.

------------------------------------------------------------------------

# Part 3 --- Does the Frontend Refresh Every 15 Minutes?

## 10. Important Correction

A common beginner assumption is:

> "If the access token lasts 15 minutes, the frontend must call
> `/refresh` every 15 minutes."

Not necessarily.

The access token simply becomes invalid after 15 minutes.

There are different ways to trigger refresh.

A simple and practical approach is:

``` text
API Request
     |
     v
Access token valid?
    / \
  YES  NO
   |    |
   v    v
Success 401
         |
         v
    /auth/refresh
         |
         v
    New access token
         |
         v
    Retry request
```

This is called a re-authentication/retry pattern.

------------------------------------------------------------------------

# 11. Example

User logs in:

``` text
10:00
```

Access token expires:

``` text
10:15
```

But the user does nothing between 10:00 and 10:25.

Nothing needs to happen at exactly 10:15.

At 10:25, the user opens the orders page.

Frontend sends:

``` http
GET /api/orders
Authorization: Bearer OLD_ACCESS_TOKEN
```

The server checks the token.

It is expired.

Server returns:

``` http
401 Unauthorized
```

The frontend authentication layer then sends:

``` http
POST /api/auth/refresh
```

The browser automatically sends the HttpOnly refresh cookie.

If the refresh token is valid:

``` text
New Access Token
```

is returned.

The frontend updates its access token and retries:

``` http
GET /api/orders
Authorization: Bearer NEW_ACCESS_TOKEN
```

The user gets:

``` text
200 OK
```

The user does not need to log in again.

------------------------------------------------------------------------

# 12. Who Triggers Refresh?

The frontend triggers the refresh request.

But you should not put refresh logic inside every React component.

Bad approach:

``` js
// Component A
if (tokenExpired) refresh();

// Component B
if (tokenExpired) refresh();

// Component C
if (tokenExpired) refresh();
```

This creates duplicated and difficult-to-maintain authentication logic.

Instead, centralize it in your API layer.

Since this practice project uses RTK Query, the ideal place is a custom
`baseQuery` wrapper.

Conceptually:

``` text
React Component
      |
      v
RTK Query
      |
      v
baseQueryWithReauth
      |
      v
API Server
```

The API layer handles:

``` text
Request
  |
  v
401?
  |
  +---- NO ---> return response
  |
  +---- YES --> refresh
                  |
                  v
             new access token
                  |
                  v
             retry request
```

------------------------------------------------------------------------

# Part 4 --- Refresh Token Rotation

## 13. What Is Refresh Token Rotation?

Suppose the browser currently has:

``` text
Refresh Token A
```

The server receives:

``` http
POST /api/auth/refresh
```

The server validates A.

Instead of continuing to use A forever, it creates:

``` text
Refresh Token B
```

Then:

``` text
A → invalid
B → valid
```

This is refresh-token rotation.

------------------------------------------------------------------------

# 14. Why Rotate Refresh Tokens?

Suppose Refresh Token A is stolen.

Without rotation:

``` text
Attacker
   |
   v
Refresh Token A
   |
   v
/refresh
   |
   v
New Access Token
```

The attacker may continue refreshing as long as A remains valid.

With rotation:

``` text
User
 |
 v
Refresh Token A
 |
 v
/refresh
 |
 +----> A becomes invalid
 |
 +----> B becomes valid
```

If an attacker later tries A:

``` text
Attacker
   |
   v
Refresh Token A
   |
   v
/refresh
   |
   v
REJECTED
```

Rotation reduces the usefulness of a stolen refresh token after it has
already been consumed.

------------------------------------------------------------------------

# 15. How Does Rotation Work With Our Session Model?

Our session record is:

``` js
{
  userId,
  refreshTokenHash,
  ip,
  userAgent,
  revoke
}
```

We do not have to create a new session record for every 15-minute
access-token refresh.

Instead, one session can represent one login/device.

Example:

``` text
Session #123

userId: user123
refreshTokenHash: hash(A)
revoke: false
```

After rotation:

``` text
Session #123

userId: user123
refreshTokenHash: hash(B)
revoke: false
```

We replaced the stored refresh-token hash.

So:

``` text
A = invalid
B = valid
```

This avoids creating unnecessary session rows every time the access
token is refreshed.

------------------------------------------------------------------------

# Part 5 --- What About Old Refresh Tokens?

## 16. Do We Need to Delete Every Old Refresh Token?

If your design replaces:

``` text
hash(A)
```

with:

``` text
hash(B)
```

then there is no old refresh-token hash to keep in that session record.

You are not accumulating:

``` text
A
B
C
D
E
F
...
```

inside the same session.

You simply maintain the current refresh-token hash.

Therefore:

``` text
Before refresh:

Session
  refreshTokenHash = hash(A)

After refresh:

Session
  refreshTokenHash = hash(B)
```

The old token A is automatically rejected because its hash is no longer
stored.

------------------------------------------------------------------------

# 17. What Does `revoke` Mean?

Our model has:

``` js
revoke: false
```

This means the session is currently active.

When the user logs out:

``` js
revoke: true
```

Now the refresh token associated with that session must no longer be
accepted.

So the server checks:

``` text
Session exists?
    |
    v
revoke === false?
    |
    v
Refresh token matches?
    |
    v
Refresh token still valid?
```

All checks must pass.

------------------------------------------------------------------------

# 18. Why Not Delete the Session During Logout?

You could delete it.

But keeping:

``` js
revoke: true
```

can be useful because you still have a record of the session.

For this beginner practice project, keeping the revoked session is
easier to understand.

Later, you can introduce cleanup policies.

For example:

``` text
Active session
      |
      v
revoke = false

Logout
      |
      v
revoke = true

After retention period
      |
      v
Delete old session
```

------------------------------------------------------------------------

# Part 6 --- Multiple Devices

## 19. Why Do We Need Sessions?

Suppose the same user logs in from:

``` text
Chrome desktop
Android phone
Firefox laptop
```

You can create:

``` text
Session #1
userId = 123
userAgent = Chrome
revoke = false

Session #2
userId = 123
userAgent = Android
revoke = false

Session #3
userId = 123
userAgent = Firefox
revoke = false
```

Now each login/device can have its own session.

If the user logs out from their phone:

``` text
Session #2
revoke = true
```

The desktop session can remain active.

This is one of the major benefits of server-side session records.

------------------------------------------------------------------------

# Part 7 --- Logout

## 20. Logout Lifecycle

User clicks:

``` text
Logout
```

Frontend:

``` http
POST /api/auth/logout
```

Browser sends the refresh cookie.

Server identifies the session and changes:

``` js
revoke: false
```

to:

``` js
revoke: true
```

Then the server clears the refresh cookie.

Conceptually:

``` text
Browser
  |
  v
POST /logout
  |
  v
Server
  |
  +--> revoke session
  |
  +--> clear refresh cookie
  |
  v
Response
```

The access token may still technically exist until its short expiration
time.

Therefore, do not design your system as if logout instantly destroys an
already-issued stateless access token unless you also implement
server-side access-token revocation.

For a simple JWT architecture, the usual approach is to keep access
tokens short-lived.

------------------------------------------------------------------------

# Part 8 --- Complete Authentication Flow

## 21. Register

``` text
Frontend
   |
   v
POST /api/auth/register
   |
   v
Validate input
   |
   v
Check existing user
   |
   v
Hash password
   |
   v
Create user
   |
   v
Response
```

------------------------------------------------------------------------

# 22. Login

``` text
Frontend
   |
   v
POST /api/auth/login
   |
   v
Validate email/password
   |
   v
Create access token
   |
   v
Create refresh token
   |
   v
Hash refresh token
   |
   v
Create session
   |
   +----> MongoDB
   |
   +----> HttpOnly cookie
   |
   v
Return access token
```

------------------------------------------------------------------------

# 23. Protected Request

``` text
Frontend
   |
   v
GET /api/users/me
   |
   v
Authorization: Bearer ACCESS_TOKEN
   |
   v
Auth middleware
   |
   v
Verify access token
   |
   +---- invalid ---> 401
   |
   +---- valid -----> controller
```

------------------------------------------------------------------------

# 24. Refresh

``` text
Frontend
   |
   v
POST /api/auth/refresh
   |
   v
Refresh cookie
   |
   v
Hash token
   |
   v
Find session
   |
   v
Check:
  - session exists
  - revoke === false
  - token matches
   |
   v
Generate new access token
   |
   v
Generate new refresh token
   |
   v
Replace refreshTokenHash
   |
   v
Set new HttpOnly cookie
   |
   v
Return new access token
```

------------------------------------------------------------------------

# 25. Logout

``` text
Frontend
   |
   v
POST /api/auth/logout
   |
   v
Find session
   |
   v
revoke = true
   |
   v
Clear cookie
   |
   v
Success
```

------------------------------------------------------------------------

# Part 9 --- Practice Project

# 26. Project Goal

Build a small authentication API.

Do not add:

-   Google login
-   Facebook login
-   Firebase
-   Clerk
-   Password reset
-   Email verification
-   Roles
-   Permissions
-   Complex service layers

The goal is to understand the authentication lifecycle first.

------------------------------------------------------------------------

# 27. Tech Stack

Use:

``` text
Node.js
Express
MongoDB
Mongoose
JWT
bcrypt
cookie-parser
Zod or another validation library
```

Frontend:

``` text
React
Redux Toolkit
RTK Query
```

------------------------------------------------------------------------

# 28. Project Structure

Keep this project intentionally simple.

``` text
auth-practice/
│
├── server/
│   │
│   ├── config/
│   │   └── db.js
│   │
│   ├── models/
│   │   ├── User.js
│   │   └── Session.js
│   │
│   ├── middleware/
│   │   └── authMiddleware.js
│   │
│   ├── routes/
│   │   └── authRoutes.js
│   │
│   ├── utils/
│   │   └── token.js
│   │
│   ├── app.js
│   └── server.js
│
└── client/
    └── React application
```

For this practice project, it is completely fine to keep the business
logic and database calls directly inside the controllers/routes.

The goal is learning the authentication lifecycle, not practicing
enterprise folder architecture.

------------------------------------------------------------------------

# 29. User Model

Create a simple user model:

``` js
import mongoose from "mongoose";

const userSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: true,
      trim: true,
    },

    email: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
      trim: true,
    },

    password: {
      type: String,
      required: true,
    },
  },
  {
    timestamps: true,
  }
);

export const User = mongoose.model("User", userSchema);
```

Important:

The password must never be stored as plain text.

Instead:

``` text
Plain password
      |
      v
bcrypt
      |
      v
Password hash
      |
      v
MongoDB
```

------------------------------------------------------------------------

# 30. Session Model

Create the session model:

``` js
import mongoose from "mongoose";

const sessionSchema = new mongoose.Schema(
  {
    userId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: "User",
      required: true,
    },

    refreshTokenHash: {
      type: String,
      required: true,
    },

    ip: {
      type: String,
    },

    userAgent: {
      type: String,
    },

    revoke: {
      type: Boolean,
      default: false,
    },
  },
  {
    timestamps: true,
  }
);

export const Session = mongoose.model("Session", sessionSchema);
```

This gives you:

``` text
userId
refreshTokenHash
ip
userAgent
revoke
createdAt
updatedAt
```

------------------------------------------------------------------------

# 31. Token Utility

Create a utility for generating tokens.

Access token:

``` js
import jwt from "jsonwebtoken";

export const createAccessToken = (userId) => {
  return jwt.sign(
    {
      userId,
    },
    process.env.ACCESS_TOKEN_SECRET,
    {
      expiresIn: "15m",
    }
  );
};
```

For the refresh token, you can generate a cryptographically secure
random value.

For example:

``` js
import crypto from "crypto";

export const createRefreshToken = () => {
  return crypto.randomBytes(64).toString("hex");
};
```

Then hash it before storing it.

Example:

``` js
export const hashRefreshToken = (token) => {
  return crypto
    .createHash("sha256")
    .update(token)
    .digest("hex");
};
```

------------------------------------------------------------------------

# 32. Register Route

Endpoint:

``` http
POST /api/auth/register
```

Expected body:

``` json
{
  "name": "Sabbir",
  "email": "sabbir@example.com",
  "password": "password123"
}
```

Flow:

``` text
Receive request
      |
      v
Validate input
      |
      v
Check email
      |
      v
Hash password
      |
      v
Create user
      |
      v
Return success
```

For this practice project, registration does not need to automatically
log the user in.

------------------------------------------------------------------------

# 33. Login Route

Endpoint:

``` http
POST /api/auth/login
```

Expected body:

``` json
{
  "email": "sabbir@example.com",
  "password": "password123"
}
```

Flow:

``` text
Find user
    |
    v
Compare password
    |
    v
Create access token
    |
    v
Create refresh token
    |
    v
Hash refresh token
    |
    v
Create Session
    |
    v
Set refresh cookie
    |
    v
Return access token
```

Example session creation:

``` js
const refreshToken = createRefreshToken();
const refreshTokenHash = hashRefreshToken(refreshToken);

await Session.create({
  userId: user._id,
  refreshTokenHash,
  ip: req.ip,
  userAgent: req.get("user-agent"),
  revoke: false,
});
```

Set the cookie:

``` js
res.cookie("refreshToken", refreshToken, {
  httpOnly: true,
  secure: process.env.NODE_ENV === "production",
  sameSite: "lax",
  path: "/api/auth",
});
```

Return:

``` json
{
  "message": "Login successful",
  "accessToken": "..."
}
```

------------------------------------------------------------------------

# 34. Authentication Middleware

Create middleware that protects routes using the access token.

Example:

``` js
import jwt from "jsonwebtoken";

export const authMiddleware = (req, res, next) => {
  const authorization = req.headers.authorization;

  if (!authorization?.startsWith("Bearer ")) {
    return res.status(401).json({
      message: "Access token required",
    });
  }

  const token = authorization.split(" ")[1];

  try {
    const payload = jwt.verify(
      token,
      process.env.ACCESS_TOKEN_SECRET
    );

    req.userId = payload.userId;

    next();
  } catch (error) {
    return res.status(401).json({
      message: "Invalid or expired access token",
    });
  }
};
```

Now a protected route can use:

``` js
router.get("/me", authMiddleware, async (req, res) => {
  const user = await User.findById(req.userId).select("-password");

  res.json({
    user,
  });
});
```

------------------------------------------------------------------------

# 35. Refresh Route

Endpoint:

``` http
POST /api/auth/refresh
```

The browser automatically sends:

``` text
HttpOnly refreshToken cookie
```

The server:

``` text
Get cookie
    |
    v
Hash refresh token
    |
    v
Find session
    |
    v
Does session exist?
    |
    +---- NO ---> 401
    |
    v
Is revoke false?
    |
    +---- NO ---> 401
    |
    v
Create new refresh token
    |
    v
Replace refreshTokenHash
    |
    v
Create new access token
    |
    v
Set new refresh cookie
    |
    v
Return access token
```

Example:

``` js
router.post("/refresh", async (req, res) => {
  const oldRefreshToken = req.cookies.refreshToken;

  if (!oldRefreshToken) {
    return res.status(401).json({
      message: "Refresh token required",
    });
  }

  const oldRefreshTokenHash =
    hashRefreshToken(oldRefreshToken);

  const session = await Session.findOne({
    refreshTokenHash: oldRefreshTokenHash,
    revoke: false,
  });

  if (!session) {
    return res.status(401).json({
      message: "Invalid refresh token",
    });
  }

  const newRefreshToken = createRefreshToken();
  const newRefreshTokenHash =
    hashRefreshToken(newRefreshToken);

  session.refreshTokenHash = newRefreshTokenHash;
  await session.save();

  const accessToken = createAccessToken(session.userId);

  res.cookie("refreshToken", newRefreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    sameSite: "lax",
    path: "/api/auth",
  });

  return res.json({
    accessToken,
  });
});
```

Notice something important:

We did not create another session.

We updated the existing session:

``` text
hash(A)
   |
   v
hash(B)
```

------------------------------------------------------------------------

# 36. Logout Route

Endpoint:

``` http
POST /api/auth/logout
```

The server gets the refresh token from the cookie.

It finds the session.

Then:

``` js
session.revoke = true;
await session.save();
```

Finally:

``` js
res.clearCookie("refreshToken", {
  httpOnly: true,
  secure: process.env.NODE_ENV === "production",
  sameSite: "lax",
  path: "/api/auth",
});
```

Return:

``` json
{
  "message": "Logout successful"
}
```

------------------------------------------------------------------------

# 37. Auth Routes

Your router can contain:

``` text
POST /register
POST /login
POST /refresh
POST /logout
```

Example:

``` js
import express from "express";

const router = express.Router();

router.post("/register", ...);

router.post("/login", ...);

router.post("/refresh", ...);

router.post("/logout", ...);

export default router;
```

Then mount it:

``` js
app.use("/api/auth", authRoutes);
```

Your final endpoints become:

``` text
POST /api/auth/register
POST /api/auth/login
POST /api/auth/refresh
POST /api/auth/logout
```

------------------------------------------------------------------------

# Part 10 --- Frontend Practice

## 38. Frontend Authentication State

Your frontend needs to know:

``` text
Is the user logged in?
Who is the user?
What is the current access token?
```

For this practice project:

``` text
Access token
    |
    v
Redux state / memory

Refresh token
    |
    v
HttpOnly cookie
```

Do not attempt to read the HttpOnly refresh cookie using JavaScript.

------------------------------------------------------------------------

# 39. RTK Query Request Flow

Create a base API:

``` js
const baseQuery = fetchBaseQuery({
  baseUrl: "http://localhost:5000/api",
  credentials: "include",
  prepareHeaders: (headers, { getState }) => {
    const token = getState().auth.accessToken;

    if (token) {
      headers.set(
        "Authorization",
        `Bearer ${token}`
      );
    }

    return headers;
  },
});
```

`credentials: "include"` is important when the frontend and backend
require cookies to be included.

------------------------------------------------------------------------

# 40. Reauthentication Logic

Your custom base query can conceptually work like this:

``` js
const baseQueryWithReauth = async (
  args,
  api,
  extraOptions
) => {
  let result = await baseQuery(
    args,
    api,
    extraOptions
  );

  if (result.error?.status === 401) {
    const refreshResult = await baseQuery(
      {
        url: "/auth/refresh",
        method: "POST",
      },
      api,
      extraOptions
    );

    if (refreshResult.data?.accessToken) {
      // Store the new access token.

      // Retry the original request.
      result = await baseQuery(
        args,
        api,
        extraOptions
      );
    } else {
      // Refresh failed.
      // Clear authentication state.
    }
  }

  return result;
};
```

This is the core idea.

The actual implementation should also handle concurrency carefully once
you move beyond a simple practice project.

------------------------------------------------------------------------

# 41. The Complete Frontend Flow

Suppose the user opens:

``` text
Orders Page
```

The component calls:

``` js
useGetOrdersQuery();
```

RTK Query sends:

``` http
GET /api/orders
Authorization: Bearer ACCESS_TOKEN
```

If valid:

``` text
200
```

If expired:

``` text
401
```

RTK Query then:

``` text
401
 |
 v
POST /api/auth/refresh
 |
 v
HttpOnly refresh cookie
 |
 v
New access token
 |
 v
Save access token
 |
 v
Retry original request
 |
 v
GET /api/orders
 |
 v
200
```

The component itself does not need to know all these details.

------------------------------------------------------------------------

# Part 11 --- Complete Example Timeline

## 42. 10:00 --- Login

``` text
Access Token A
expires = 10:15

Refresh Token R1
expires = 7 days
```

Database:

``` js
{
  userId: "123",
  refreshTokenHash: hash(R1),
  ip: "...",
  userAgent: "...",
  revoke: false
}
```

Browser:

``` text
Access Token A → memory
Refresh Token R1 → HttpOnly cookie
```

------------------------------------------------------------------------

## 43. 10:05 --- API Request

``` http
GET /api/profile
Authorization: Bearer A
```

A is valid.

``` text
200 OK
```

No refresh is needed.

------------------------------------------------------------------------

## 44. 10:16 --- API Request

A has expired.

``` http
GET /api/profile
Authorization: Bearer A
```

Server:

``` text
401 Unauthorized
```

Frontend:

``` http
POST /api/auth/refresh
```

Browser automatically sends:

``` text
R1
```

Server validates R1.

Then generates:

``` text
Access Token B
Refresh Token R2
```

Database becomes:

``` js
{
  userId: "123",
  refreshTokenHash: hash(R2),
  revoke: false
}
```

Browser:

``` text
Access Token B → memory
Refresh Token R2 → HttpOnly cookie
```

Frontend retries:

``` http
GET /api/profile
Authorization: Bearer B
```

Result:

``` text
200 OK
```

------------------------------------------------------------------------

# 45. 10:30 --- Another Request

The frontend continues using:

``` text
Access Token B
```

No refresh is required until B expires or the application otherwise
decides to refresh.

------------------------------------------------------------------------

# 46. Seven Days Later

If the refresh session has expired or is otherwise no longer valid:

``` http
POST /api/auth/refresh
```

fails:

``` text
401 Unauthorized
```

The frontend should clear authentication state and send the user to the
login page.

------------------------------------------------------------------------

# 47. User Logs Out

``` text
POST /api/auth/logout
```

Server:

``` text
session.revoke = true
```

Cookie:

``` text
refreshToken = cleared
```

Now the refresh session cannot be used.

------------------------------------------------------------------------

# Part 12 --- Security Rules for This Practice Project

## 48. Rule #1 --- Hash Passwords

Never:

``` js
password: "12345678"
```

Store:

``` text
bcrypt hash
```

------------------------------------------------------------------------

## 49. Rule #2 --- Never Store Raw Refresh Tokens

Store:

``` text
hash(refreshToken)
```

not:

``` text
refreshToken
```

------------------------------------------------------------------------

## 50. Rule #3 --- Keep Access Tokens Short-Lived

For practice:

``` text
15 minutes
```

is a good example.

------------------------------------------------------------------------

## 51. Rule #4 --- Protect Refresh Tokens

Use:

``` text
HttpOnly
Secure
SameSite
```

cookies.

------------------------------------------------------------------------

## 52. Rule #5 --- Validate Refresh Sessions

Do not simply accept any refresh token.

Check:

``` text
Session exists
AND
revoke === false
AND
hash matches
```

------------------------------------------------------------------------

## 53. Rule #6 --- Do Not Put Refresh Logic Everywhere

Do not write refresh logic in every component.

Centralize it in your API/authentication layer.

For your stack, RTK Query's base-query layer is a good place.

------------------------------------------------------------------------

# Part 13 --- Beginner Mental Model

If all of this feels complicated, remember only this:

``` text
LOGIN
 |
 +---- Access Token
 |       15 minutes
 |
 +---- Refresh Token
         7 days
         |
         v
     HttpOnly Cookie
```

Then:

``` text
Access Token expires
        |
        v
Protected request fails with 401
        |
        v
Frontend calls /refresh
        |
        v
Server validates refresh token
        |
        v
Rotate refresh token
        |
        v
Return new access token
        |
        v
Frontend retries request
```

And the session:

``` text
Session
├── userId
├── refreshTokenHash
├── ip
├── userAgent
├── revoke
├── createdAt
└── updatedAt
```

That's the entire system.

------------------------------------------------------------------------

# Part 14 --- Practice Project Checklist

Build the project in this order.

## Step 1 --- Setup

-   [ ] Create Express server
-   [ ] Connect MongoDB
-   [ ] Configure environment variables
-   [ ] Add cookie-parser
-   [ ] Add JWT
-   [ ] Add bcrypt
-   [ ] Add CORS if frontend is separate

## Step 2 --- User

-   [ ] Create User model
-   [ ] Register user
-   [ ] Hash password
-   [ ] Prevent duplicate email

## Step 3 --- Login

-   [ ] Verify email
-   [ ] Verify password
-   [ ] Generate access token
-   [ ] Generate refresh token
-   [ ] Hash refresh token
-   [ ] Create session
-   [ ] Set HttpOnly cookie
-   [ ] Return access token

## Step 4 --- Protected Route

-   [ ] Create auth middleware
-   [ ] Read Bearer token
-   [ ] Verify JWT
-   [ ] Attach userId to request
-   [ ] Create `/me`

## Step 5 --- Refresh

-   [ ] Read refresh cookie
-   [ ] Hash refresh token
-   [ ] Find session
-   [ ] Check `revoke`
-   [ ] Generate new access token
-   [ ] Generate new refresh token
-   [ ] Replace `refreshTokenHash`
-   [ ] Set new cookie
-   [ ] Return access token

## Step 6 --- Logout

-   [ ] Find session
-   [ ] Set `revoke = true`
-   [ ] Clear cookie
-   [ ] Return success

## Step 7 --- Frontend

-   [ ] Store access token in application state
-   [ ] Send access token with API requests
-   [ ] Detect 401
-   [ ] Call `/refresh`
-   [ ] Store new access token
-   [ ] Retry failed request
-   [ ] Clear auth state when refresh fails

------------------------------------------------------------------------

# Part 15 --- Testing Scenarios

Do not stop after getting login to work.

Test these cases manually.

### Test 1 --- Normal Login

``` text
Login
→ access token returned
→ refresh cookie created
→ session created
```

### Test 2 --- Protected Route

``` text
Valid access token
→ /me
→ 200
```

### Test 3 --- Expired Access Token

``` text
Expired access token
→ /me
→ 401
```

### Test 4 --- Refresh

``` text
Expired access token
→ /refresh
→ new access token
→ new refresh token
```

### Test 5 --- Old Refresh Token

After rotation:

``` text
Old refresh token
→ /refresh
→ rejected
```

### Test 6 --- Logout

``` text
Logout
→ revoke = true
→ cookie cleared
```

Then:

``` text
Refresh
→ rejected
```

### Test 7 --- Multiple Sessions

Login from two browsers.

You should see:

``` text
Session A → revoke false
Session B → revoke false
```

Logout from one session:

``` text
Session A → revoke true
Session B → revoke false
```

The second session should remain active.

------------------------------------------------------------------------

# Final Mental Model

Do not think of authentication as:

``` text
"JWT = login"
```

Think of it as a lifecycle:

``` text
                    LOGIN
                      |
          +-----------+-----------+
          |                       |
          v                       v
    ACCESS TOKEN            REFRESH TOKEN
      15 minutes               7 days
          |                       |
          |                       v
          |                 HttpOnly Cookie
          |                       |
          v                       v
    Protected APIs          Session Record
          |                       |
          |                       |
          +----------+------------+
                     |
                     v
              Access expires
                     |
                     v
                  401
                     |
                     v
                /refresh
                     |
                     v
             Validate Session
                     |
                     v
              Rotate Refresh
                     |
                     v
             New Access Token
                     |
                     v
              Retry API request
                     |
                     v
                   200
```

The most important thing to remember is:

> **The access token is for accessing APIs. The refresh token is for
> obtaining a new access token. The session record gives the server
> control over the long-lived authentication session.**

And for the practice architecture:

``` text
User
├── name
├── email
└── passwordHash

Session
├── userId
├── refreshTokenHash
├── ip
├── userAgent
├── revoke
├── createdAt
└── updatedAt
```

This is deliberately simpler than a production enterprise authentication
system. That is a feature, not a weakness: first understand the
lifecycle, then add complexity such as refresh-token reuse detection,
session/device management, concurrent-refresh locking, password reset,
email verification, rate limiting, CSRF protection, and role/permission
systems.
