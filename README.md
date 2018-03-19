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

Using our cross-platform SDKs, all chat data is sent through our hosted API where we manage chat state and broadcast it to your clients:

![](https://i.imgur.com/qybeCr6.jpg)

You'll never have to worry about scale or infrastructure, we take care of it all for you.

Perhaps the best way to learn Chatkit is to start building, so I highly reccomend you follow along. Along the way, you'll learn best practices when using Chatkit with React.

Alright, let's code!

## Step 1. Download the starter template

Rather than start from absoloute scratch, this walkthrough is based on a minimal starter template:

As you will see, the starter termplate doesn't contain any interesting logic, just boilerplate we need to run a React application and a simple Node server. 

Our client-side is based on the [Create React App template](https://github.com/facebook/create-react-app). The server uses Node. If you're unfamiliar with Node, don't worry. After the next section, we won't really touch the server.

To get started, download the starter template then run `npm install`:

```
git clone
cd ./
npm install
```

(This tutorial assumes the use of `npm`, but the equivalent `yarn` commands will work as well.)

## Step 2. Create a Chatkit instance

Like I illustrated earlier, all chat data is sent through and managed by a _Chatkit instance_.

To create a Chatkit instance, head to the dashboard, hit **Create new**, then give your instance a name. I will call mine “React Chat Tutorial”:

In the Keys tab, take note of the **Instance Locator** and **Keys**. We'll need these both in the next section.



## Step 3. Setup the server

While _most_ interactions will happen on the client, Chatkit also needs a server component to create and manage users securely:

![](https://i.imgur.com/9elZ5SQ.jpg)

We won't authorise users in this tutorial, but we'll still need to define a route (like `/users`) that, when called, creates a Chatkit user.

Start by installing [`pusher-chatkit-server`](https://www.npmjs.com/package/pusher-chatkit-server):

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

* First, we import `Chatkit` from `pusher-chatkit-server`
* Then, instantiate our own `chatkit` instance using the **Insance Locator** and **Key** from the dashboard.
* Before a user can connect to Chatkit, a Chatkit user must first be created. In the `/users` route, we take a `username` supplied by the client and create a Chatkit user. We'll call this route directly in the next step.
* Authentication is the action of proving a user is who she says she is. When someone first connects to Chatkit, a request will be sent to `/authenticate` to authenticate her. The server needs to respond with a token (returned by `chatkit.authenticate`) if the user is valid. In our case, we will (naïvely) assume everyone is who they say they are, and return a token from `chatkit.authenticate` rno matter what.

That's all we need to don the server! Let's move on to the client.

## Step 4. Login 

When someone loasds the app we want to ask them who they are:

Once they hit Submit we will send their username to the server to createa a Chatkit user if one doesn't exist. 

To collect their username, createa UsernameForm in src/components:

```diff
+import React, { Component } from 'react'

+class UsernameForm extends Component {
+ constructor(props) {
+   super(props)
+   this.state = {
+     username: '',
+   }
+   this.onSubmit = this.onSubmit.bind(this)
+   this.onChange = this.onChange.bind(this)
+ }

+ onSubmit(e) {
+   e.preventDefault()
+   this.props.onSubmit(this.state.username)
+ }

+ onChange(e) {
+    this.setState({ username: e.target.value })
+  }
+
+  render() {
+    return (
+      <div>
+        <div>
+          <h2>What is your usernane?</h2>
+          <form onSubmit={this.onSubmit}>
+            <input
+              type="text"
+              placeholder="Your full name"
+              onChange={this.onChange}
+            />
+            <input type="submit" />
+          </form>
+        </div>
+      </div>
+    )
+  }
+}
+
+ export default UsernameForm
```

Then, update App.js:

```diff
import React, { Component } from 'react'
+import UsernameForm from './components/UsernameForm'

class App extends Component {
  constructor() {
    super()
+    this.state = {
+      currentScreen: 'WhatIsYourUsernameScreen',
+      currentUsername: '',
+    }
+    this.onUsernameSubmitted = this.onUsernameSubmitted.bind(this)
  }

+  onUsernameSubmitted(username) {
+    fetch('http://localhost:3001/users', {
+      method: 'POST',
+      headers: {
+        'Content-Type': 'application/json',
+      },
+      body: JSON.stringify({ username }),
+    })
+      .then(response => {
+        this.setState({
+          userId: username,
+          currentScreen: 'ChatScreen',
+        })
+      })
+      .catch(error => console.error('error', error))
+  }

  render() {
-     return <h1>Chat</h1>
+    if (this.state.currentScreen === 'WhatIsYourUsernameScreen') {
+      return <UsernameForm onSubmit={this.onUsernameSubmitted} />
+    }
  }
}
+
export default App
```





