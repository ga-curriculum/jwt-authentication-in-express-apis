# ![Authenticating an Express API with JWT - Sign-In Route](./assets/hero.png)

**Learning objective:** By the end of this lesson, students will be able to Implement a sign in route with express and JWTs.

```js
// controllers/users.js

router.post('/signin', async (req, res) => {
    try {
        const user = await User.findOne({ username: req.body.username });
        if (user && bcrypt.compareSync(req.body.password, user.hashedPassword)) {
            const token = jwt.sign({ username: user.username }, process.env.JWT_SECRET);
            res.status(200).json({ token });
        } else {
            res.status(401).json({ error: 'Invalid username or password.' });
        }
    } catch (error) {
        res.status(400).json({ error: error.message });
    }
});
```





<!-- 
Create the route that accepts username and password as POST request inputs and returns either an unauthorized response with generic "credentials are invalid" error message, or a signed jwt authorizing the user into the app. -->
