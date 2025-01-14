<h1>
  <span class="headline">JWT Authentication in Express APIs</span>
  <span class="subhead">Protecting Routes</span>
</h1>

**Learning objective:** By the end of this lesson, students will be able to use a token payload to protect a route from unauthorized users.

## Protecting routes

At this stage, we're authenticating users when they sign in to our app by providing them a token to include in future requests. Next, we want to use those tokens to protect specific routes from unauthenticated or unauthorized users.

We could write code to handle this in each controller function attached to a protected route, but that would not be very DRY.

Instead, let's create middleware that can be used on any route requiring authentication before proceeding.

First, create a middleware folder and a `verify-token` js file:

```bash
mkdir middleware
touch middleware/verify-token.js
```

In the Setting up JWTs lesson, we wrote a function that accepts a token (via req.headers) and then verifies it:

```javascript
// controllers/test-jwt.js

router.post('/verify-token', (req, res) => {
  try {
    const token = req.headers.authorization.split(' ')[1];
    const decoded = jwt.verify(token, process.env.JWT_SECRET);

    res.json({ decoded });
  } catch (err) {
    res.status(403).json({ err: 'Invalid token.' });
  }
});
```

The good news is that this function will work perfectly as a middleware, with a few minor changes. Instead of sending a response to a successful token, the function should pass the request on to the `next` response handler.

This means adding a third parameter in addition to `req` and `res` - `next` - which will let us invoke the next function in the request-response cycle, just like any other middleware function.

Secondly, we'll want to change what we do with the decoded payload in the token. In our test function above, we responded to the request with the decoded payload. Instead, we'll want to store information about the authenticated user on the `request` object before passing that request along to its next function.

That way, any controller function can tell what user a request is coming from, allowing for user-specific data queries and authorization of certain actions, among other things. We'll assign the token's `decoded.payload` to `req.user` - the `verifyToken` function is acting as intermediary middleware, so we don't need to respond with anything yet.

Finally, we call `next()`:

```javascript
// middleware/verify-token.js

// We'll need to import jwt to use the verify method
const jwt = require('jsonwebtoken');

function verifyToken(req, res, next) {
  try {
    const token = req.headers.authorization.split(' ')[1];
    const decoded = jwt.verify(token, process.env.JWT_SECRET);

    // Assign decoded payload to req.user
    req.user = decoded.payload;

    // Call next() to invoke the next middleware function
    next();
  } catch (err) {
    // If any errors, send back a 401 status and an 'Invalid token.' error message
    res.status(401).json({ err: 'Invalid token.' });
  }
}

// We'll need to export this function to use it in our controller files
module.exports = verifyToken;
```

## Using the `verifyToken()` middleware

We'll implement two routes to test our middleware - one that protects against unauthenticated users, and another that protects against both unauthenticated and unauthorized users. Both routes will interact with our `User` model. So, let's create a `controllers/users.js` file to handle these new routes.

```bash
touch controllers/users.js
```

Our application's user data is pretty dull without extra details added to the user model, so a more robust application might want more details (or even a second, one-to-one Profile model) attached to each user.

Our first route will only protect against unauthenticated users using the `verifyToken()` middleware. It will show a list of all users in the database. Here are the specs for the route:

- Route: `/users`
- Method: `GET`
- Response Body: `[ { username : <user's username>, _id : <user's _id> } ]`

Let's spin up an unprotected version of this route:

```javascript
// controllers/users.js

const express = require('express');
const router = express.Router();

const User = require('../models/user');

router.get('/', async (req, res) => {
  try {
    // Get a list of all users, but only return their username and _id
    const users = await User.find({}, "username");

    res.json(users);
  } catch (err) {
    res.status(500).json({ err: err.message });
  }
});

module.exports = router;
```

Now, test this route by sending a `GET` request to `http://localhost:3000/users`. You should be able to get a response even when sending a request through your browser, which has no token and is, therefore, unauthenticated. This means our route is open to anyone.

## Authenticating on the users route

Now that we've set up our `verify-token` middleware, including it in the users route will take a small adjustment:

```javascript
// controllers/users.js

// Add to your imports at the top
const verifyToken = require('../middleware/verify-token');

router.get('/', verifyToken, async (req, res) => {
  // The rest of the function remains unchanged
});
```

When a request hits this route, `verifyToken()` is called before the anonymous controller function that handles the response. As a result, only authenticated users can access these routes; a non-verified user will be returned an `Invalid token.` message, the request would not get past the `verifyToken()` middleware function, and the controller function would never be invoked.

Test the route again. Access denied!

Now enable the same Authorization options in Postman for your `http://localhost:3000/users` route. Select `Bearer Token` as the type and paste in the token given when the user with that `_id` posts to the `signin` route.

You're in!

## Protecting a user's details from other users

A classic use case for authentication and authorization is the profile page of any web application, showing the user their personal details and allowing them to make updates to their account. Since no one else should be able to access this, we should protect it from unauthenticated **and** unauthorized users. These are the specs of the route we want:

- Route: `/users/:userId`
- Method: `GET`
- Response Body: `{ username : <user's username>, _id : <user's _id> }`

Let's start with the following unprotected route that will let anyone at all access the details of any user:

```javascript
// controllers/users.js

const express = require('express');
const router = express.Router();
const User = require('../models/user');

router.get('/:userId', async (req, res) => {
  try {
    const user = await User.findById(req.params.userId);
    if (!user) {
      return res.status(404).json({ err: 'User not found.'});
    }

    res.json({ user });
  } catch (err) {
    res.status(500).json({ err: err.message });
  }
});

module.exports = router;
```

Now, test this route at GET `http://localhost:3000/users/:userId` with the `_id` of one of your app's created users. You should be able to get a response even when sending a request through your browser.

> 💡 If you're wondering where to find the `_id` of a user, the logged-in user's `_id` should still be stored in the token you used for the `Bearer <token>` header in Postman, so you could access that `_id` by making an HTTP request to our `/test-jwt/verify-token` route from earlier. You can also directly access the database through Mongo Atlas.

## Authenticating the user's show route

Let's add the `verifyToken` middleware to the user's show route so that only authenticated users can access it:

```javascript
// controllers/users.js

router.get('/:userId', verifyToken, async (req, res) => {
  // The rest of the function remains unchanged
});
```

Again, you'll be denied until you enable the same Authorization options in Postman for your `http://localhost:3000/users/:userId` route. Select `Bearer Token` as the type and paste in the token given when the user with that `_id` posts to the `signin` route.

Test this again, and you should be able to access any user's details.

## Authorization

For many routes, authentication is enough. The dashboard of most web applications will be accessible to any signed-in user but will reject requests from someone not yet signed in.

This user route, however, is also interested in **authorizing** the user and only allowing access if the request comes from that user.

Fortunately, our `verifyToken` middleware stores the `_id` of the request's user in the `req` object, so we can make another adjustment to the top of our controller function:

```javascript
// controllers/users.js

router.get('/:userId', verifyToken, async (req, res) => {
  try {
    // If the user is looking for the details of another user, block the request
    // Send a 403 status code to indicate that the user is unauthorized
    if (req.user._id !== req.params.userId){
      return res.status(403).json({ err: "Unauthorized"});
    }

    const user = await User.findById(req.params.userId);

    if (!user) {
      return res.status(404).json({ err: 'User not found.'});
    }

    res.json({ user });
  } catch (err) {
    res.status(500).json({ err: err.message });
  }
});
```

There we have it! This strategy applies to any case where you want to restrict actions to the user who owns a specific resource, like editing or deleting their posts on social media.

A more sophisticated authorization approach would grant users specific roles and permission levels, allowing an application's administrators to have access to any user's resources.

## Conclusion

We now have an Express app that authenticates and authorizes users through a JWT approach. Consider it the start of a re-usable boilerplate for any Express API that needs user management. However, this isn't the full picture of user management. For a complete system, you'd want to add email verification, password resets, oAuth integration, passkey support, and other key features expected in applications managing users.

Besides, there are plenty of authentication management libraries in the JavaScript ecosystem, so creating your own from scratch with JWT, as we've done here, is hardly necessary. At their foundation, however, all token-based authentication libraries and APIs employ the same strategy we've implemented in this lesson, and being comfortable with the management and exchange of these tokens is critical no matter which token authentication implementation you're working with.
