# ![Authenticating an Express API with JWT - Protecting Routes](./assets/hero.png)

**Learning objective:** By the end of this lesson, students will be able to use a token payload to protect a route from unauthorized users.

```js
// middleware/verify-token.js

const jwt = require('jsonwebtoken');

function verifyToken(req, res, next) {
    const token = req.headers.authorization.split(' ')[1];
    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = decoded;
        next();
    } catch (error) {
        res.status(401).json({ error: 'Invalid token.' });
    }
}

module.exports = verifyToken;
```


```js
// controllers/profiles.js

const express = require('express');
const router = express.Router();
const Profile = require('../models/profile');
// Import our middleware: 
const verifyToken = require('../middleware/verify-token');

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



<!-- Implement a route accessible only to signed-in users, and possibly a route specific to an individual user (i.e. profiles/:userId) that gets validated by the token's payload.

Return an unauthorized JSON response to non-authenticated or unauthorized users. -->