# Build a Chat App with React and Pusher Chatkit

In this tutorial, you’ll learn how to build a chat app with React and [Chatkit](https://pusher.com/chatkit). When we're done, we'll have a chat application complete with **Typing indicators**, a **"Who's online" list**, and **message history**: 

<ANIMATION OF FINAL DEMO>
  
  

If you think this sounds like a lot to tackle in one tutorial... You would normally be right! However, because we're going to use [Chatkit](pusher.com/chatkit), we can more or less focus exclusively on the front-end React code while Chatkit does the heavy lifting.

## Prerequisities

* Before delving into the walkthrough, it would be good to have a basic understanding of [React](https://reactjs.org/tutorial/tutorial.html)


## What is Chatkit?

[Chatkit](pusher.com/chatkit) is a hosted API that helps you build impressive chat features into your applications with less code. Features like,

* Group chat
* One-to-one chat
* Typing indicators
* "Who's online" presence
* Read receipts
* File transfers

Using our cross-platform SDKs, all chat data is sent through the hosted API where we manage chat state and broadcast it to your clients:

![](https://i.imgur.com/qybeCr6.jpg)

You'll never have to worry about scale or infrastructure, we take care of it all for you.

Perhaps the best way to learn Chatkit is to start building, so I highly reccomend you follow along. Along the way, you'll learn best practices when using Chatkit with React.


## Setting up the developer environment

Rather than start from absoloute scratch, this walkthrough is based on a minimal starter template:

The starter termplate doesn't contain any interesting logic, just boilerplate we need to run a React application and a simple Node server. 

Our client-side is based on the [Create React App template](https://github.com/facebook/create-react-app). The server uses Node

If you're unfamiliar with Node, don't worry. After the next section, we won't really touch the server.

To get started, download the starter template then run `npm install`:

```
git clone
cd ./
npm install
```

(This tutorial assumes the use of `npm`, but the equivalent `yarn` commands will work as well.)


## Setting up the server

While _most_ interactions will happen on the client, Chatkit also needs a server component to create and manage users securely:

![](https://i.imgur.com/9elZ5SQ.jpg)

We won't authorise users in this tutorial but we'll still need a route to create a Chatkit user.

Start by installing `pusher-chatkit-server`:

```
npm install --save pusher-chatkit-sever
```

Then update `server.js`:

```diff
const express = require('express')
const bodyParser = require('body-parser')
const cors = require('cors')
+const Chatkit = require('pusher-chatkit-server')

const app = express()

+const chatkit = new Chatkit.default({
+  instanceLocator: 'YOUR INSTANCE LOCATOR',
+  key: 'YOUR KEY',
+})

app.use(bodyParser.urlencoded({ extended: false }))
app.use(bodyParser.json())
app.use(cors())

+app.post('/users', (req, res) => {
+  const { username } = req.body
+  chatkit
+    .createUser(username, username)
+    .then(() => res.sendStatus(201))
+    .catch(error => {
+      if (error.error_type === 'services/chatkit/user/user_already_exists') {
+ res.sendStatus(200)
+      } else {
+        res.status(error.statusCode).json(error)
+      }
+    })
+})

+app.post('/authenticate', (req, res) => {
+  const grant_type = req.body.grant_type
+  res.json(chatkit.authenticate({ grant_type }, req.query.user_id))
+})

const PORT = 3001
app.listen(PORT, err => {
  if (err) {
    console.error(err)
  } else {
    console.log(`Running on port ${PORT}`)
  }
})
```

There's a lot to unpack here, starting from the top:

* First, import `pusher-chatkit-server`
* Then, instantiate a `chatkit` instance using your unique **Insance Locator** and **Key** 
* Before a user can connect to Chatkit, a Chatkit user must be created. In the `/users` route, take a `username` and create a Chatkit user. We'll call this route directly from React in an upcoming step.
* Authentication is the action of proving a user is who she says she is. When someone first connects to Chatkit, a request will be sent to `/authenticate` to authenticate her. The server needs to respond with a token (returned by `chatkit.authenticate`) _if_ the user is valid. In our case, we are going to assume everyone is who they say they are and return a token from `chatkit.authenticate` no matter what. 
