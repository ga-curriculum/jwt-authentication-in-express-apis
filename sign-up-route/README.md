# ![[tktk Module Name] - tktk Microlesson Name](./assets/hero.png)

**Learning objective:** By the end of this lesson, students will be able to tktk

## User Model

In this section, we will be setting up an Express application to use JSON Web Tokens (JWT) for **authentication** and **authorization**. For this, we will need a user model to store user information. We will use Mongoose to create a user model with the following fields:

| Field          | Type   | Description            |
| -------------- | ------ | ---------------------- |
| username       | String | User's username        |
| hashedPassword | String | User's hashed password |

### tktk hat emoji You do: Create a User Model

The goal of this exercise is to create a user model using Mongoose. You will need to create a new file called `user.js` in a `models` directory and define the user schema and model.

The user schema should have the following fields:

- `username`: A string that is required and unique.
- `hashedPassword`: A string that is required.

Make sure to create and export the user model at the end of the file.

## Sign Up Route

Let's create a route to allow users to sign up for our application. This route will be responsible for creating a new user in the database. For this step we are going to be using `bcrypt` to hash the user's password before saving it to the database. 

Create a new file in the `controllers` folder called `users.js`. Inside of `users.js` we want set up a new router and create a POST route for `/sign-up`. This route will be responsible for creating a new user in the database, but for now, let's send back a message to the user to let them know that the route is working.

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

We never want to save passwords dirrectly to the database. This is because if our database gets hacked, people's passwords would be exposed to the hackers. This could lead to other accounts being stolen that use the same password.

Bcrypt is a package that will encrypt passwords for us so we are not saving them dirrectly to the database. It's not important to know exactly how Bcrypt works but let's go over a high overview of what it's doing to our passwords.

1. **Hashes the Password**: Applies a one-way hash function to our passwords that converts them into a fixed-length string of characters. This is a mathematical process that produces a unique output for each input, making it practically impossilbe to reverse-engineer the original password from the hash.
2. **Salting**: Uses a technique called "salting". This is were random string of characters are added to the password before hashing it. This random string of characters is often refered to as the "salt". Adding the "salt" to a password ensures if two users have the same password they do not end up with the same hash.
3. **Adjustable Work Factor**: Allows you to adjust the "work factor" or the number of times a password is hashed. However this will increase the time the hash function runs for. As we increase the number of times we hash we dirrectly increase the time the function will need to run for.

There are two main methods in the Bcrypt library that allows us to hash a passwords.

- `hash()`: Used for **asynchronously** generating a hash for the given string. It returns promise is callback is committed and you need to resolve the promise.
- `hashSync()`: Used to **synchronously** generates a hash for the given string. It returns the hashed string

We are going to use the `hashSync()` method for our hashing purposes.

Let's install Bcrypt and require it in our `controllers/users.js` file, along with the user model.

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

Inside of our `singup` route, remove the placeholder `res.json`, add `async` to the start of the callback function and include a `try...catch` block.

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

Using the `create` method on the user model, we are going to create a new user. For the field `hashedPassword` we want to called the Bcrypt method `hashSync`. This method intakes two arguments:

- The password to be hashed. Or string to be hashed.
- The salt length to generate.

Once we create the new user we will want to send the status of `201` created along with the user to the client.

```js
// controllers/users.js
// ... require statements above

// Add in constant for the number
const SALT_LENGTH = 12;

router.post('/signup', async (req, res) => {
    try {
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

The last things we have to do for this route is handle any errors that might happen. Let's do that by setting the status code to `400` and sending the error message to the client.

```js
// controllers/users.js
// ... require statements and constant above
router.post('/signup', async (req, res) => {
    try {
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

Test this by using Postman to make a POST request to `http://localhost:3000/users/signup`. Make sure to send in the correct body in the request. It should be formatted as such: 

```json
{
    "username": "<insert test username>",
    "password": "<insert test password>"
}
```

We should recieve a reponse that includes the `hashedPassword`. While this is not our plain text password we shouldn't be sending this inside of a JSON object. It should live safely inside of our database. Let's fix this. 

## Remove Hased Password from Response

To remove the hashed password from our JSON response we will need to add a function to our Schema. In Mongoose there is a method called `set` where we set options for our Schemas. There is an option in our Schemas that is called `toJSON` and this is called when we take an instance of a document and send it through a function or method, like `res.json()`, that transfroms the docment to JSON. 

This is perfect because we want to remove the field `hashedPassword` when we send it to the client using `res.json()`. Let's implement this. Inside of `models/user.js` and under where we created our `userSchema` let's call the `set` method on our `userSchema`.

```js
// models/user.js
// ... userSchema above

userSchema.set();
```

The `set` method is going to intake two arguments:

- Key: *String* - The name of the option to set.
- Value: *Object* - The value of the option we want to set.

We already know we want to change the `toJSON` option so that will be the key we want to change. For the value we want to delete the field `hashedPassword` from the document. To to this we will use *transform*. Transform is expecting a callback function and it will hand that callback function two items. First the original document from the database, the second will be what is returned when the function is run.

Inside of the `set` method let's add the two arguments.

```js
// models/user.js
// ... userSchema above
userSchema.set('toJSON', {
    transform: (document, returnedObject) => {
        delete returnedObject.hashedPassword;
    }
});
```

Test again with Postman and you can see that the hashed password has been removed.
