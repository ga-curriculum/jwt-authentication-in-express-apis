# ![JWT Authentication in Express APIs - Setup](./assets/hero.png)

## Setup

Open your Terminal application and navigate to your `~/code/ga/lectures` directory:

```bash
cd ~/code/ga/lectures
```

Make a new directory called `express-api-jwt-auth-template`, then enter this directory:

```bash
mkdir express-api-jwt-auth-template
cd express-api-jwt-auth-template
```

Create `server.js`, `.env`, and `.gitignore` files:

```bash
touch server.js .env .gitignore
```

In the terminal, create a `package.json` with all of the default settings and install the packages by running the following commands:

```bash
npm init -y
npm i express mongoose dotenv
```

Open the contents of the directory in VS Code:

```bash
code .
```

Fill in the `.gitignore` file with the following:

```
node_modules
.env
```

Add the following boilerplate code to the `server.js` file:

```js
const dotenv = require('dotenv');
dotenv.config();
const express = require('express');
const app = express();
const mongoose = require('mongoose');

mongoose.connect(process.env.MONGODB_URI);

mongoose.connection.on('connected', () => {
  console.log(`Connected to MongoDB ${mongoose.connection.name}.`);
});

app.use(express.json());

// Routes go here

app.listen(3000, () => {
  console.log('The express app is ready!');
});
```

Add your MongoDB URI to the `.env` file:

```bash
MONGODB_URI=mongodb+srv://<username>:<password>@sei-w0kys.azure.mongodb.net/express-api-jwt-auth?retryWrites=true
```

> If you are unsure of where to obtain your MongoDB URI, please refer to the MongoDB Atlas Setup Lab.

Finally, run the server with `nodemon`:

```bash
nodemon server.js
```

We will be using Postman during this lesson to interact with our API. If you do not have Postman installed, please refer to the Postman Setup Lab.
