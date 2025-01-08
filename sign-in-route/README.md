<h1>
  <span class="headline">JWT Authentication in Express APIs</span>
  <span class="subhead">Sign-In Route</span>
</h1>

**Learning objective:** By the end of this lesson, students will be able to implement a sign-in route with express and JWTs.

## Sign in route

Now that a user can sign up, let's create a route that allows users to sign in to our app. As with the sign-up route, we'll use bcrypt to check the user's credentials. Start by framing the route in the `auth.js` file with a simple response and error handling:

```javascript
// controllers/auth.js

router.post('/sign-in', async (req, res) => {
  try {
    res.status(200).json({ message: 'Signing in!' });
  } catch (err) {
    res.status(500).json({ err: err.message });
  }
});
```

We'll start by confirming the user has supplied a valid username. If the username is not found in the database, we'll return a 401 status code and a message that the credentials are invalid.

```javascript
// controllers/auth.js

router.post('/sign-in', async (req, res) => {
  try {
    // Look up the user by their username in the database
    const user = await User.findOne({ username: req.body.username });
    // If the user doesn't exist, return a 401 status code with a message
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials.' });
    }

    res.status(200).json({ message: 'Signing in!' });
  } catch (err) {
    res.status(500).json({ err: err.message });
  }
});
```

If the user is found, we'll check if the password is correct.

We'll use the bcrypt's `compareSync()` method to check the user's password. This method takes two arguments - a plain text password and a hashed password - and checks if they match. If so, the method returns `true`. Otherwise, it will return `false` - we'll store that in a variable called `isPasswordCorrect`.

If the passwords don't match, we'll return a 401 status code and a message that the credentials are invalid, just like we did when the username didn't exist.

> 🧠 Returning a generic message when the user has submitted a bad username or password is considered a better security practice than sending back a unique message specific to the invalid field.

```javascript
// controllers/auth.js

router.post('/sign-in', async (req, res) => {
  try {
    const user = await User.findOne({ username: req.body.username });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials.' });
    }

    // Check if the password is correct using bcrypt
    const isPasswordCorrect = bcrypt.compareSync(
      req.body.password, user.hashedPassword
    );
    // If the password is incorrect, return a 401 status code with a message
    if (!isPasswordCorrect) {
      return res.status(401).json({ message: 'Invalid credentials.' });
    }

    res.status(200).json({ message: 'Signing in!' });
  } catch (err) {
    res.status(500).json({ err: err.message });
  }
});
```

Create a new user and save those credentials to test your `sign-in` route.

Using Postman, make a POST request to `http://localhost:3000/auth/sign-in` and check if you get the expected response.

## Issue a token

Next, we want to return a token to the client if they are authorized to sign in. This means our function will look identical to the sign-up route from here.

We'll create a new `payload` variable and assign it to the data we want to send as the token's payload. Then, we'll make the token and send it `token` to the client.

```javascript
// controllers/auth.js

router.post('/sign-in', async (req, res) => {
  try {
    const user = await User.findOne({ username: req.body.username });
    if (!user) {
      return res.status(401).json({ message: 'Invalid credentials.' });
    }

    const isPasswordCorrect = bcrypt.compareSync(
      req.body.password, user.hashedPassword
    );
    if (!isPasswordCorrect) {
      return res.status(401).json({ message: 'Invalid credentials.' });
    }

    // Construct the payload
    const payload = { username: user.username, _id: user._id };

    // Create the token, attaching the payload
    const token = jwt.sign({ payload }, process.env.JWT_SECRET);

    // Send the token instead of the message
    res.status(200).json({ token });
  } catch (err) {
    res.status(500).json({ err: err.message });
  }
});
```

> ⚠️ Note that the payload is identical to the one we used in the sign-up route. We typically want to keep the payload consistent across our routes to ensure that the data the client receives is identical whether the user just signed up or is signing in for the hundredth time.

Fantastic! We now have a working sign-in route that will check the user's credentials and then create and supply a JWT back to the user if their password matches the hashed password on the server.

Next, we'll work on using the token to protect routes in our app!
