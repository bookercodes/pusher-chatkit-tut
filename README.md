# Build a Chat App with React and Pusher Chatkit

## Introduction

In this tutorial, you’ll learn how to build a chat app with React and [Pusher Chatkit](https://pusher.com/chatkit). When we're done we'll have a chat complete with **Typing indicators**, a **"Who's online" list**, and **message history**: 

<ANIMATION OF FINAL DEMO>
  
  

If you think this sounds like a lot to tackle in one tutorial... You would normally be right. However, because we're going to be using [Pusher Chatkit](pusher.com/chatkit), we can more or less focus exclisively on the front-end React code while Chatkit does the heavy lifting on the back-end.

Before delving into the walkthrough, it would be good to have a basic understanding of [React](https://reactjs.org/tutorial/tutorial.html). The main focus of this tutorial is Chatkit.

## What is Chatkit?







## Sever Setup

In the template , you'll find a file called `server.js` which spins up a simple Express sever. If you're familiar with Express,it should all be pretty familiar.


body-parser
cors
express




* Why we need server
* What it will do

In the the project root, install  `pusher-chatkit-server`:

```
npm install --save pusher-chatkit-server
```

Then, update `server.js`:

```diff
const express = require('express')
const bodyParser = require('body-parser')
+const Chatkit = require('pusher-chatkit-server')
const cors = require('cors')

const app = express()

+const chatkit = new Chatkit.default({
+  instanceLocator: 'YOUR INSTANCE LOCATOR',
+  key: 'YOUR KEY',
+ })

app.use(bodyParser.urlencoded({ extended: false }))
app.use(bodyParser.json())
app.use(cors())

+ app.post('/users', (req, res) => {
+  const { username } = req.body
+  chatkit
+    .createUser(username, username)
+    .then(() => res.sendStatus(201))
+    .catch(error => {
+      if (error.error_type === 'services/chatkit/user/user_already_exists') {
+        res.sendStatus(200)
+      } else {
+        res.status(error.statusCode).json(error)
+      }
+    })
+ })


+app.post('/authenticate', (req, res) => {
+  const { grant_type}  = req.body
+  const { user_id } = req.query
+  res.json(chatkit.authenticate({ grant_type }, user_id))
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


There's a fair amount to unpack here, starting from the top:

* First, we import [`pusher-chatkit-server`](https://www.npmjs.com/package/pusher-chatkit-server), the Node Chatkit SDK
* Then, we instantiate our own `Chatkit` instance using the *Instance Locator* and *Key* from the previous step. Remember to replace "YOUR INSTANCE LOCATOR" and "YOUR KEY" with your own respective values
* 
