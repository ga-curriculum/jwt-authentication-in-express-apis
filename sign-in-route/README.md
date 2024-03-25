# ![Authenticating an Express API with JWT - Sign In Route](./assets/hero.png)

**Learning objective:** By the end of this lesson, students will be able to implement a sign in route with express and JWTs.

## Sign In Route

Now that a user can sign up, next let's create a route that allows users to sign in to our application. As with the sign up route, we'll be using `bcrypt` - this time to handle checking the user's credentials. 


We'll use the built in Bcrypt `compareSync()` method to check the user's password. This method takes two arguments - an unhashed password and a hashed password - and checks if they match. If so, the method returns `true`, otherwise it will return `false`. 

For now, we'll just return a boilerplate message in the instance that the passwords match, return a different message for invalid credentials, and add our standard error handling in the `catch` block:  

```js
// controllers/users.js

router.post('/signin', async (req, res) => {
    try {
        const user = await User.findOne({ username: req.body.username });
        if (user && bcrypt.compareSync(req.body.password, user.hashedPassword)) {
            res.json({ message: 'You are authorized!' });
        } else {
            res.json({ message: "Invalid credentials."})
        }
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});
```

Next, we want to send a token back to the client in the instance that they are authorised to sign in. First, we'll have to import `jwt` into this file, the same way we did with the `test-jwt` controller. We'll create a new `token` variable, and assign it to the result of signing the token by passing the valid user's username as the payload, and the `JWT_SECRET` from our `.env` file as the secret key. Finally, we send this `token` to the client, giving it a status of `200` (OK). 

If the `compareSync()` method returns false, we want to send a 401 status, specifying in the error message that the supplied username or password were incorrect.

```js
// controllers/users.js
// At the top of the file
const jwt = require('jsonwebtoken');

// other routes here

router.post('/signin', async (req, res) => {
    try {
        const user = await User.findOne({ username: req.body.username });
        if (user && bcrypt.compareSync(req.body.password, user.hashedPassword)) {
            const token = jwt.sign({ username: user.username, _id: user._id }, process.env.JWT_SECRET);
            res.status(200).json({ token });
        } else {
            res.status(401).json({ error: 'Invalid username or password.' });
        }
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});
```

Fantastic! We now have a working Sign In route that will check the user's credentials, and then create and supply a JWT back to the user if their password matches the hashed password on the server. 

## Refactoring the Sign Up Process

In most applications, signing up for a new account will automatically authenticate the new user. Let's implement that by adding our fancy new `jwt` code to the response from our `/users/signup` route, and including the token in our response as well:

```js
router.post('/signup', async (req, res) => {
    try {
        // Check if the username is already taken
        const userInDatabase = await User.findOne({ username: req.body.username });
        if (userInDatabase) {
            return res.json({error: 'Username already taken.'});
        }
        // Create a new user with hashed password
        const user = await User.create({
            username: req.body.username,
            hashedPassword: bcrypt.hashSync(req.body.password, SALT_LENGTH)
        })
        const token = jwt.sign({ username: user.username, _id: user._id }, process.env.JWT_SECRET);
        res.status(201).json({ user, token });
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});
```

Next, we'll work on using the token to protect routes in our app! 