# ![Authenticating an Express API with JWT - Sign Up Route](./assets/tktk-hero.png)

**Learning objective:** By the end of this lesson, students will be able to implement a sign up route with Express and JWTs.

## User Model

This section will set up an Express application to use JSON Web Tokens (JWT) for **authentication** and **authorization**. For this, we will need a user model to store user information. We'll use Mongoose to create a user model with the following fields:

| Field          | Type   | Description            |
| -------------- | ------ | ---------------------- |
| username       | String | User's username        |
| hashedPassword | String | User's hashed password |

### 🎓 You do: Create a User Model

The goal of this exercise is to create a user model using Mongoose. Create a new file called `user.js` in a `models` directory and define the user schema and model.

The user schema should have the following fields:

- `username`: A string that is required and unique.
- `hashedPassword`: A string that is required.

Make sure to create and export the user model at the end of the file.

## Sign Up Route

Next let's create a route to allow users to sign up for our application. This route will be responsible for creating a new user in the database. For this step, we will use `bcrypt` to hash the user's password before saving it to the database. 

Create a new file in the `controllers` folder called `users.js`. Inside `users.js`, we want to set up a new router and create a POST route for `/signup`. This route will be responsible for creating a new user in the database, but for now, let's just send back a message to the user to let them know that the route is working.

```js
// /controllers/users.js
const express = require('express');
const router = express.Router();

router.post('/signup', (req, res) => {
    res.json({ message: 'Signup route' });
});

module.exports = router;
```

Require and use the route inside of `server.js`:

```js
// server.js

// ... other imports
const testJWTRouter = require('./controllers/test-jwt');
const usersRouter = require('./controllers/users');

// ... other middleware

// Routes go here
app.use('/test-jwt', testJWTRouter);
app.use('/users', usersRouter);
```

Using Postman, make a POST request to `http://localhost:3000/users/signup` and check if you get the expected response.

### Bcrypt

We never want to save passwords directly to the database. This is because if our database gets hacked people's passwords would be exposed to the hackers which could lead to other accounts using the same password being stolen.

Bcrypt is a package that will encrypt passwords for us, so that we are not saving them directly to the database. Knowing precisely how Bcrypt works is not essential, but let's review a high-level overview of what it's doing to our passwords:

1. **Hashes the Password**: Applies a one-way hash function to our passwords, converting them into a fixed-length string of characters. This mathematical process produces a unique output for each input, making it practically impossible to reverse-engineer the original password from the hash.
2. **Salting**: "Salting" is where a random string of characters is added to the password before hashing it. This random string of characters is often called the "salt" - adding the "salt" to a password ensures that if two users have the same password, they do not end up with the same hash.
3. **Adjustable Work Factor**: You can adjust the "work factor" or the number of times a password is hashed. However, this will increase the time the hash function runs. As we increase the number of times we hash, we directly increase the time the function will need to run.

Two main methods in the Bcrypt library allow us to hash passwords.

- `hash()`: Used for **asynchronously** generating a hash for the given string. It returns a promise that would need to be resolved.
- `hashSync()`: Used to **synchronously** generates a hash for the given string. It returns the hashed string.

We will use the `hashSync()` method for our hashing purposes.

Let's install Bcrypt and require it in our `controllers/users.js` file and the user model.

```bash
npm i bcrypt
```

```js
// controllers/users.js
const express = require('express');
const router = express.Router();
// Add bcrypt and the user model
const bcrypt = require('bcrypt');
const User = require('../models/user');

// ... route still below
```

Inside our `signup` route, remove the placeholder `res.json`, add `async` to the start of the callback function, and include a `try...catch` block: 

```js
// controllers/users.js
// ... require statements above
router.post('/signup', async (req, res) => {
    try {
        // Set up for try
    } catch (error) {
        // Set up for catch
    }
});
```

We will create a new user using the `create` method on the user model. For the field `hashedPassword`, we want to call the Bcrypt method `hashSync`. This method intakes two arguments:

- The password to be hashed. 
- The salt length to generate.

Once we create the new user, we will want to send the status of `201` created along with the user to the client. 

Of course, it's also vital that we don't end up with two users with the same username, so we'll also add a database call checking whether the username is already taken before creating a new one.

```js
// controllers/users.js
// ... require statements above

// Add in constant for the number
const SALT_LENGTH = 12;

router.post('/signup', async (req, res) => {
    try {
        // Check if the username is already taken
        const userInDatabase = await User.findOne({ username: req.body.username });
        if (userInDatabase) {
            return res.status(400).json({error:'Username already taken.'});
        }
        // Create a new user with hashed password
        const user = await User.create({
            username: req.body.username,
            hashedPassword: bcrypt.hashSync(req.body.password, SALT_LENGTH)
        })
        res.status(201).json({ user });
    } catch (error) {
        // Set up for catch
    }
});
```

The last thing we have to do for this route is handle any errors that might happen. Let's do that by setting the status code to `400` and sending the error message to the client.

```js
// controllers/users.js
// ... require statements and constant above
router.post('/signup', async (req, res) => {
    try {
        // Check if the username is already taken
        const userInDatabase = await User.findOne({ username: req.body.username });
        if (userInDatabase) {
            return res.status(400).json({error:'Username already taken.'});
        }
        // Create a new user with hashed password
        const user = await User.create({
            username: req.body.username,
            hashedPassword: bcrypt.hashSync(req.body.password, SALT_LENGTH)
        })
        res.status(201).json({ user });
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});
```

Test this using Postman to make a POST request to `http://localhost:3000/users/signup`. Make sure to send the correct body in the request. It should be formatted as such: 

```json
{
    "username": "<insert test username>",
    "password": "<insert test password>"
}
```

We should receive a response that includes the `hashedPassword`. While this is not our plain text password, we still shouldn't send it inside a JSON object. It should live safely inside our database. Let's fix that next. 

## Remove Hashed Password from Response

We will need to add a function to our Schema to remove the hashed password from our JSON response.

In Mongoose, there is a method called `set` where we set options for our Schemas. There is an option in our Schemas called `toJSON`, which is called when we take an instance of a document and send it through a function or method, like `res.json()`, that transforms the document to JSON. This is perfect because we want to remove the field `hashedPassword` when we send it to the client using `res.json()`. 

Inside of `models/user.js` and where we created our `userSchema`, let's call the `set` method on our `userSchema`.

```js
// models/user.js
// ... userSchema above

userSchema.set();
```

The `set` method is going to intake two arguments:

- Key: *String* - The option's name to set.
- Value: *Object* - The option's value we want to set.

We already know we want to change the `toJSON` option, so that will be the key we want to change. For the value, we want to delete the field `hashedPassword` from the document. To do this, we will use *transform*. `transform` expects a callback function, and it will pass that callback function two arguments. 

1. The original document from the database.
2. What is returned when the function is run.

Inside the `set` method, let's add the two arguments.

```js
// models/user.js
// ... userSchema above
userSchema.set('toJSON', {
    transform: (document, returnedObject) => {
        delete returnedObject.hashedPassword;
    }
});
```

Test again with Postman, and you can see that the hashed password has been removed.
