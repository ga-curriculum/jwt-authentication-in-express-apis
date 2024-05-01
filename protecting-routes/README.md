# ![JWT Authentication in Express APIs - Protecting Routes](./assets/hero.png)

**Learning objective:** By the end of this lesson, students will be able to use a token payload to protect a route from unauthorized users.

## Protecting routes

At this stage, we're authenticating users when they sign in to our app by providing them a token to include with their future requests. Next, we want to use those tokens to protect specific routes from unauthenticated or unauthorized users.

We could write code to handle this in each controller function that is attached to a protected route, but that would not be very DRY.

Instead, let's create middleware that can be used on any route that should require authentication before proceeding.

First, create a middleware folder and a `verify-token` js file:

```bash
mkdir middleware
touch middleware/verify-token.js
```

In the Setting up JWTs lesson, we wrote a function that accepts a token (via req.headers) and then verifies it:

```js
// controllers/test-jwt.js

router.post('/verify-token', (req, res) => {
  try {
    const token = req.headers.authorization.split(' ')[1];
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    res.json({ decoded });
  } catch (error) {
    res.status(401).json({ error: 'Invalid token.' });
  }
});
```

The good news is that this function will work perfectly as a middleware, with a few small changes. Instead of sending a response to a successful token, the function should simply pass the request on to the `next` response handler.

This means adding a third parameter in addition to `req` and `res` - `next` - which will let us invoke the next function in the request-response cycle, just like any other middleware function.

Secondly, we'll want to change what we are doing with the decoded payload contained in the token. In our test function above, we responded to the request with the decoded payload. Now, we instead want to store information about the authenticated user on the `request` object before passing that request along to its next function. That way, any controller function will be able to tell what user a request is coming from, allowing for user-specific data queries and authorization of certain actions, among other things. We'll assign the token's `decoded` to `req.user` - the `verifyToken` function is acting as intermediary middleware, so we don't need to respond with anything yet.

After attaching `decoded` to `req`, we call `next()`:

```js
// middleware/verify-token.js

// We'll need to import jwt to use the verify method
const jwt = require('jsonwebtoken');

function verifyToken(req, res, next) {
  try {
    const token = req.headers.authorization.split(' ')[1];
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    // Assign decoded payload to req.user
    req.user = decoded;
    // Call next() to invoke the next middleware function
    next();
  } catch (error) {
    // If any errors, send back a 401 status and an 'Invalid token.' error message
    res.status(401).json({ error: 'Invalid token.' });
  }
}

// We'll need to export this function so that we can use it in our controllers file
module.exports = verifyToken;
```

## Using the verifyToken middleware

A classic use case for authentication and authorization is the profile page of any web application, showing the user their personal details and allowing them to make updates to their account. Since no one else should be able to access this, we should protect it from unauthenticated **and** unauthorized users.

Let's create a `controllers/profiles.js` file to handle this particular route in a new controller.

Of course, our application's profile data is pretty boring without extra details added to the user model, so a more robust application might want more details (or even a second, one-to-one Profile model) attached to each user.

Regardless, these are the specs of the route we want:

- Route: `/profiles/:userId`
- Method: `GET`
- Response Body: `{ user: { username : <user's username>, _id : <user's _id> } }`

Let's start with the following, unprotected code that will let anyone at all access the details of any user:

```js
// controllers/profiles.js
const express = require('express');
const router = express.Router();
const User = require('../models/user');

router.get('/:userId', async (req, res) => {
  try {
    const user = await User.findById(req.params.userId);
    if (!user) {
      res.status(404);
      throw new Error('Profile not found.');
    }
    res.json({ user });
  } catch (error) {
    if (res.statusCode === 404) {
      res.status(404).json({ error: error.message });
    } else {
      res.status(500).json({ error: error.message });
    }
  }
});

module.exports = router;
```

Be sure to also plug this into `server.js` with an `app.use` statement:

```js
// at the top, with other imports
const profilesRouter = require('./controllers/profiles');

// ... other middleware

// Routes go here
app.use('/profiles', profilesRouter);
```

Now, test this route at GET `http://localhost:3000/profiles/<user-id>` with the `_id` of one of your app's created users. You should be able to get a response even when sending a request through your browser, which has no token and is therefore unauthenticated. This means our route is open to anyone, unauthenticated and with no authorization.

If you're wondering where to find the `_id` of a user, the logged-in user's `_id` should still be stored in the token you used for the `Bearer <token>` header in Postman, so you could access that `_id` by making an HTTP request to our `/test-jwt/verify-token` route from earlier. Or, just directly access the database through Mongo Atlas.

## Authenticating the profiles route

Now that we've set up our `verify-token` middleware, including it in the profile route will be simple:

```javascript
// controllers/profiles.js

// add to your imports at the top
const verifyToken = require('../middleware/verify-token');

router.get('/:userId', verifyToken, async (req, res) => {
  // rest of the function remains unchanged
});
```

Now, when a request hits this route, `verifyToken` is called before the anonymous controller function that handles the response. As a result, only authenticated users will be able to access these routes; a non-verified user will be returned an `Invalid token.` message, the request would not get past the `verifyToken` middleware function, and the controller function will never be invoked at all.

Test the route again. Access denied!

Now enable the same Authorization options in Postman for your `http://localhost:3000/profiles/:userid` route. Select `Bearer Token` as the type and paste in the token given when the user with that `_id` posts to the `signin` route.

You're in!

## Authorization

For many routes, authentication is enough. The dashboard of most web applications will be accessible to any signed-in user, but will reject requests from someone not yet signed in.

This profile route, however, is also interested in **authorizing** the user, and only allowing access if the request is coming from the user who owns that profile.

Fortunately, our `verifyToken` middleware stores the `_id` of the request's user in the `requests` object, so we can make another simple adjustment to the top of our controller function:

```javascript
// controllers/profiles.js

router.get('/:userId', verifyToken, async (req, res) => {
    try {
        if (req.user._id !== req.params.userId){ // check the ID of the user!
            return res.status(401).json({ error: "Unauthorized"})
        }
```

There we have it! This strategy is applicable to any case where you want to restrict actions to the user who owns a specific resource, like editing or deleting their posts on social media.

A more sophisticated authorization approach would grant users specific roles and permission levels, allowing an application's administrators to have access to any user's resources.

## Conclusion

We now have an Express app that authenticates and authorizes users through a JWT approach. Consider it the start of a re-usable boilerplate for any Express API that needs user management. However, this isn't the full picture of user management. For a complete system, you'd need to add email verification, password resets, oAuth integration, and other key features that are expected in applications managing users.

Besides, there are plenty of authentication management libraries in the JS ecosystem, so creating your own from scratch with JWT like we've done here is hardly necessary. At their foundation, however, all token-based authentication libraries and APIs are employing the same strategy we've implemented in this lesson, and being comfortable with the management and exchange of these tokens is critical no matter which token authentication implementation you're working with.
