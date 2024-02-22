# ![Authenticating an Express API with JWT - Protecting Routes](./assets/hero.png)

**Learning objective:** By the end of this lesson, students will be able to use a token payload to protect a route from unauthorized users.

## Protecting Routes

At this stage, we're authenticating users when they sign in to our app. Next, we want to use the tokens that the sign in process generates to protect specific routes 

We could write code to handle this in each controller function that is attached to a protected route, but that would be fairly laborious and not very DRY.

The more efficient method is to create middleware that can be used on any route that should be require authentication to hit. 

First, create a middleware folder and a `verify-token` js file: 

```bash
mkdir middleware
touch middleware/verify-token.js
```

In the Setting up JWTs lesson, we wrote a function that accepts a token (via req.headers) and then verifies it: 

```js
// controllers/test-jwt.js
router.post('/verify-token', (req, res) => {
    const token = req.headers.authorization.split(' ')[1];
    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        res.json({ decoded });
    } catch (error) {
        res.status(401).json({ error: 'Invalid token.' });
    }
});
```

The good news is that this function will work perfectly as a middleware, with a few small changes. 

First, we'll change the function from an anonymous function to a named function. We'll also add a third parameter - `next` - which will let us invoke the next function in the request-response cycle.

Secondly, we'll want to change what we are doing with the decoded payload. In our test function above, we responded to the request with the decoded payload. Now, we instead want to assign `decoded` to `res.user` - the `verifyToken` function is acting as intermediary middleware, so we don't need to respond with anything yet. 

After attaching `decoded` to `req`, we call `next()`: 

```js 
// middleware/verify-token.js

// We'll need to import jwt to use the verify method
const jwt = require('jsonwebtoken');

function verifyToken(req, res, next) {
    const token = req.headers.authorization.split(' ')[1];
    try {
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

Let's head over to our profiles controller file, and import our new middleware function: 

```js
// controllers/profiles.js

const express = require('express');
const router = express.Router();
const Profile = require('../models/profile');
// Import our middleware: 
const verifyToken = require('../middleware/verify-token');
```

Next, we'll add the `verifyToken` middleware to our route handlers: 

```js
// controllers/profiles.js

// Add middleware to route
router.post('/', verifyToken, async (req, res) => {
    try {
        const profile = await Profile.create(req.body)
        res.json({ profile })
    } catch (error) {
        res.status(500).json({ error: error.message })
    }
});

// Add middleware to route
router.get('/:userId', verifyToken, async (req, res) => {
    try {
        const profile = await Profile.findOne({ owner: req.params.userId });
        if (!profile) {
            res.status(404)
            throw new Error('Profile not found.');
        }
        res.json({ profile });
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

When the user hits a protected route, `verifyToken` is called before the anonymous controller function that handles the response. As a result, only authenticated users will be able to access these routes; a non-verified user will receive an `Invalid token.` message and the controller function will never be invoked at all. 