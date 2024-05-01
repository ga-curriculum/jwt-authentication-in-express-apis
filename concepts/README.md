# ![JWT Authentication in Express APIs - Concepts](./assets/hero.png)

**Learning objective:** By the end of this lesson, students will be able to describe using JWTs as an authentication strategy.

## Authentication and authorization

APIs often require some sort of **authentication** for you to use them - that is, some way of knowing who you are. They'll also often manage resource access, which we call authorization. We'll frequently use these two words together, but they handle two distinct parts of an application.

| Authentication                           | Authorization                              |
| ---------------------------------------- | ------------------------------------------ |
| Confirms users are who they say they are | Manages user access to different resources |

## Token authentication with JWT

There are many ways to manage **authentication**, but for this lesson we'll focus on Token Authentication. This style of authentication in HTTP uses _tokens_, or unique identifier strings, to tell who a user is when they're making an **authenticated request**.

JSON Web Tokens (JWT) are a type of token authentication. JWT utilizes a JSON-formatted token that is signed with a secret key. This token is then sent to the client, who sends it back to the server with every request. The server can then verify the token and use the information in it to **authenticate** the user.

## Anatomy of a JWT

![tktk Hunter - can we include the parts of a JWT here? something like this but way less detailed](./assets/jwt-inspo.png)

A JSON Web Token consists of three main parts:

1. **Header**: The header tells us two key things: what type of token this is (which is always JWT) and what method (algorithm) was used to secure it, like HMAC SHA256 or RSA. It's just a short piece of JSON code that has been encoded to be safe for URLs.

2. **Payload**: This part contains the claims, which are pieces of information about the user or the system that created the token. The payload can include things like the user's ID or the token's expiration time. Like the header, it's encoded to be URL-safe.

3. **Signature**: The signature is used to make sure the token hasn't been altered after it was issued and to verify that it's truly from the source it claims to be from. It's created by taking the encoded header, the encoded payload, and a secret key, and then applying the algorithm specified in the header.

Together, these parts are concatenated with dots `.` separating them, forming the compact form of the JWT: `header.payload.signature`. This structure ensures that JWTs are URL-safe, meaning they can be passed in HTTP environments without causing issues with encoding.

## Authentication flow with JWT

Let's go through the flow of how JWT works in an application.

### Token is generated

![tktk Hunter -  need visual of the server-client interaction and token generation process.]()

1. **User Login**: The user submits their credentials (username and password) to the server.
2. **Verify Credentials**: The server checks the credentials. If they are valid, it proceeds to generate a JWT.
3. **Create JWT**: The JWT is created with a JSON payload that includes user details such as user ID and username. This payload may also contain roles or permissions.
4. **Encrypt Token**: The JWT is then encrypted with a secret key that is only known to the server, resulting in a secure, encrypted token.

### Token is issued to client

![tktk Hunter -  need visual showing how the JWT is sent to the client and stored for future use.]()

1. **Issue Token**: After successful authentication, the encrypted JWT is sent back to the client.
2. **Store Token**: The client securely stores the JWT, typically in local storage or cookies, ready for future requests.

### Token is used for future requests to server

![tktk Hunter -  need visual showing the server verifying the JWT included in an incoming HTTP request.]()

1. **Include Token in Requests**: For requests to access protected resources, the client includes the stored JWT in the HTTP headers.
2. **Extract and Decrypt Token**: The server extracts the JWT from the headers, decrypts it with its secret key, and verifies its authenticity.
3. **Verify Payload**: The server reads the JSON payload to authenticate the user by verifying details such as user existence, permissions, and token expiry.
4. **Grant Access**: If the JWT is valid and the user is authenticated, the server grants access to the requested resource.

In conclusion, JWTs are an efficient, modern solution for securing web applications, providing a way to both verify users and manage their access to resources in one reusable token. Understanding JWTs helps you build web applications that are both secure and scalable.
