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
* Photo, video, and audio messages

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
* Then, instantiate our own `chatkit` instance using the **Insance Locator** and **Key** from the dashboard
* In the `/users` route, we take a `username` and create a Chatkit user through our `chatkit` instance
* Authentication is the action of proving a user is who she says she is. When someone first connects to Chatkit, a request will be sent to `/authenticate` to authenticate her. The server needs to respond with a token (returned by `chatkit.authenticate`) if the user is valid. In our case, we will (naïvely) assume everyone is who they say they are, and return a token from `chatkit.authenticate` no matter what.

Boom 💥! That's all we need to don the server. Let's move on to the client...

## Step 4. Login 

When someone loads the app, we want to ask them who they are:

Once they hit **Submit**, we'll send their username to the server (to the `/users` route we just defined) and create a a Chatkit user if one doesn't exist. 

To collect the user's name, create a file called `UsernameForm.js` in in `src/components/`:

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

Then, update `App.js`:

```diff
import React, { Component } from 'react'
+import UsernameForm from './components/UsernameForm'

class App extends Component {
  constructor() {
    super()
+    this.state = {
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
+          userId: username
+        })
+      })
+      .catch(error => console.error('error', error))
+  }

  render() {
-   return <h1>Chat</h1>
+   return <UsernameForm onSubmit={this.onUsernameSubmitted} />
  }
}

export default App
```

The `UsernameForm` component probably looks familiar to you, it's a [React Form with a controlled component](https://reactjs.org/docs/forms.html).

There isn't a whole lot to the `App` container either. We render the `UsernameForm` and, when the `onSubmit` event happens, send the username to the `/users` route, where a Chatkit user is created. If the request is successful, we update `this.state.username` so we can reference it later; otherwise, we `conosle.error` the error. 


## Step 5. Rendering the ChatScreen

When the user submits their name, we want to render the chat:

Start by creating a container called `ChatScreen` in `/src/`:

```diff
+import React, { Component } from 'react'
+
+class ChatScreen extends Component {  
+  render() {
+    const styles = {
+      container: {
+        display: 'flex'
+      }
+    }
+    return (
+      <div style={styles.container}>
+        <h1>Chat</h1>
+      </div>
+    )
+  }
+}
+
+export default ChatScreen
```

Then, update `App.js`:

```diff
import React, { Component } from 'react'
import UsernameForm from './components/UsernameForm'

class App extends Component {
  constructor() {
    super()
    this.state = {
      currentUsername: '',
+     currentScreen: 'WhatIsYourUsernameScreen' 
    }
    this.onUsernameSubmitted = this.onUsernameSubmitted.bind(this)
 }

  onUsernameSubmitted(username) {
    fetch('http://localhost:3001/users', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ username }),
    })
      .then(response => {
        this.setState({
          userId: username,
+          currentScreen: 'ChatScreen'
        })
      })
      .catch(error => console.error('error', error))
  }

 render() {
+    if (this.state.currentScreen === 'WhatIsYourUsernameScreen') {
      return <UsernameForm onSubmit={this.onUsernameSubmitted} />
+    }
+    if (this.state.currentScreen === 'Chatscreen') {
+      return <ChatScreen userId={this.state.userId} /> 
+    }
  }
}

export default App
```

Normally, you would use a router like [react-router](https://www.npmjs.com/package/react-router) to transition screens but because our app is quite simple, we update `this.state.currentScreen` and conditionally render `UsernameForm` or `ChatScreen` in the `render` function.

When we render `ChatScreen`, we pass `this.state.username` as a prop.  Now, `ChatScreen` has all the information needed to connect to Chatkit. Let's do that next...


## Step 6. Connect to Chatkit

Earlier, we installed `pusher-chatkit-server`. Now we're in client-land, you'll need to install [`pusher-chatkit-client`](https://www.npmjs.com/search?q=pusher-chatkit-client) too:

```
npm install --save pusher-chatkit-client
```

Then, update `ChatScreen.js`:

```diff
import React, { Component } from 'react'
+import Chatkit from 'pusher-chatkit-client'

class ChatScreen extends Component {
+  constructor(props) {
+    super(props)
+    this.state = {
+      currentUser: {}
+    }
+  }
  
+  componentDidMount () {
+    const chatManager = new Chatkit.ChatManager({
+      instanceLocator: 'YOUR INSTANCE LOCATOR',
+      userId: this.props.userId,
+      tokenProvider: new Chatkit.TokenProvider({
+        url: 'http://localhost:3001/authenticate',
+      }),
+    })
+
+    chatManager
+      .connect()
+      .then(currentUser => {
+        this.setState({ currentUser }) 
+     })
+     .catch(error => console.error('error', error))
+  }

  render() {
    const styles = {
      container: {
        display: 'flex'
      }
    }
    return (
      <div style={styles.container}>
        <h1>Chat</h1>
      </div>
    )
  }
}

export default ChatScreen
```

* Once the compnent has mounted, `componentDidMount` is called and we instantiate a `ChatManager` with our `instanceLocator`, `userId` (from `this.props.userId`), and a custom `TokenProvider`. The `TokenProvider` points to `/authenticate`, which we defined earlier.
* Once `ChatManager` has been instantiated, we call `connect`.
* `connect` happens asynchronously and a [`Promise`](https://developers.google.com/web/fundamentals/primers/promises) is returned. If you have followed these steps exaclty, you will connect. That being said, watch out for any `console.error`s in case you you missed something.

I am quite excited about the rest of the tutoral, and I hope you are too! Now that we have our boilerplate and a Chatkit connection, we can rapidly start to add chat features. Seriously, it's so satifying. Leggo!

## Step 7. Create a Chatkit room

In the dashboard, go to the **Inspector** and create a room: 

## Step 8. Component structure


Going forward, we'll break each feature into a indepdent (reusable, if you want!) React components:


We'll define each component as we go along but to make the tutorial a bit easier to follow, let's set out the basic page layout now by updating `ChatScreen`:




```diff
import React, { Component } from 'react'
import Chatkit from 'pusher-chatkit-client''

class ChatScreen extends Component {
  constructor(props) {
    super(props)
    this.state = {
      currentUser: {}
    }
  }

  componentDidMount () {
    const chatManager = new Chatkit.ChatManager({
      instanceLocator: 'YOUR INSTANCE LOCATOR',
      userId: this.props.userId,
      tokenProvider: new Chatkit.TokenProvider({
        url: 'http://localhost:3001/authenticate',
      }),
    })

    chatManager
      .connect()
      .then(currentUser => {
        this.setState({ currentUser })
      })
      .catch(error => console.error('error', error))
  }
  
  render() {
+    const styles = {
+      container: {
+        height: '100vh',
+        display: 'flex',
+        flexDirection: 'column',
+        color: 'white',
+      },
+      header: {
+        padding: 20,
+      },
+      chatContainer: {
+        display: 'flex',
+        flex: 1,
+      },
+      whosOnlineListContainer: {
+        width: '15%',
+        padding: 20,
+      },
+      chatListContainer: {
+        width: '85%',
+        display: 'flex',
+        flexDirection: 'column',
+      },
+      chatList: {
+        padding: 20,
+        flex: 1,
+      },
+    }
+    return (
+      <div style={styles.container}>
+        <header style={styles.header}>
+          <h2>Chatly</h2>
+        </header>
+        <div style={styles.chatContainer}>
+          <aside style={styles.whosOnlineListContainer}>
+
+          </aside>
+          <section style={styles.chatListContainer}>
+          
+          </section>
+        </div>
+      </div>
+    )
  }
}

export default ChatScreen
```

If you run the app now, you'll see the basic layout take place:






## Step 9. Subscribe to messages

I am really excited to show you this. Now we have a `Chatkit` connection, building chat features become as simple as hooking up Chatkit events to UI components. Here, let me show you.

First, create a `MessageList.js` component in `/src/components`:


```diff
+ import React, { Component } from 'react'
+ 
+ class MessagesList extends Component {
+   render() {
+     const styles = {
+       container: {
+         overflowY: 'scroll',
+       },
+       ul: {
+         listStyle: 'none',
+       },
+       li: {
+         marginTop: 13,
+         marginBottom: 13,
+       },
+       senderUsername: {
+         fontWeight: 'bold',
+       },
+       message: { fontSize: 15 },
+     }
+     return (
+       <div
+         style={{
+           ...this.props.style,
+           ...styles.container,
+         }}
+       >
+         <ul style={styles.ul}>
+           {this.props.messages.map((message, index) => (
+             <li key={index} style={styles.li}>
+               <div>
+                 <span style={styles.senderUsername}>{message.senderId}</span>{' '}
+               </div>
+               <p style={styles.message}>{message.text}</p>
+             </li>
+           ))}
+         </ul>
+       </div>
+     )
+   }
+ }
+ 
+ export default MessagesList
```


Then update `ChatScreen.js`:

```diff
import React, { Component } from 'react'
import Chatkit from 'pusher-chatkit-client'
import SendMessageForm from './components/SendMessageForm'
import WhosOnlineList from './components/WhosOnlineList'
import MessagesList from './components/MessagesList'
import TypingIndicator from './components/TypingIndicator'

class ChatScreen extends Component {
  constructor(props) {
    super(props)
    this.state = {
      currentUser: {},
+      currentRoom: {},
+      messages: []
    }
  }

  componentDidMount () {
    const chatManager = new Chatkit.ChatManager({
      instanceLocator: 'YOUR INSTANCE LOCATOR',
      userId: this.props.userId,
      tokenProvider: new Chatkit.TokenProvider({
        url: 'http://localhost:3001/authenticate',
      }),
    })

    chatManager
      .connect()
      .then(currentUser => {
        this.setState({ currentUser })
+        return currentUser.subscribeToRoom(
+          5599364,
+          {
+            newMessage: message => {
+              this.setState({
+                messages: [...this.state.messages, message],
+              })
+            },
+          },
+          100
+        )
+      })
+      .then(currentRoom => {
+        this.setState({ currentRoom })
+      })
      .catch(error => console.error('error', error))
  }

  render() {
    const styles = {
      container: {
        height: '100vh',
        display: 'flex',
        flexDirection: 'column',
        color: 'white',
      },
      header: {
        backgroundImage:
          'linear-gradient(to right, #2e646d, #2e646d, #2e646d, #2e646d, #2e646d)',
        padding: 20,
      },
      chatContainer: {
        display: 'flex',
        flex: 1,
      },
      whosOnlineListContainer: {
        width: '15%',
        backgroundColor: '#2b303b',
        backgroundImage:
          'linear-gradient(to bottom, #336f78, #2d6a79, #296579, #296079, #2b5a78)',
        padding: 20,
      },
      chatListContainer: {
        width: '85%',
        display: 'flex',
        flexDirection: 'column',
        backgroundImage:
          'linear-gradient(to bottom, #437f86, #3e7a88, #3c7689, #3c7089, #3f6b88)',
      },
+      chatList: {
+        padding: 20,
+        flex: 1,
      },
    }
    return (
      <div style={styles.container}>
        <header style={styles.header}>
          <h2>Chatly</h2>
        </header>
        <div style={styles.chatContainer}>
          <aside style={styles.whosOnlineListContainer}>
          </aside>
          <section style={styles.chatListContainer}>
+            <MessagesList
+              messages={this.state.messages}
+              style={styles.chatList}
            />
          </section>
        </div>
      </div>
    )
  }
}

export default ChatScreen

```

* Once you connect to Chatkit you get a `currentUser` object that represents... the current connected user
* Chatkit is "user-driven" meaning most if not all interactions happen on the `currentUser`
* In this case we call `subscribeToRoom` on the `currentUser` (`currentUser.subscribeToRoom`)
* `subscribeToRoom` takes an event handler called `onNewMessage` that is fired each time a new message arrives. Because we specific `100`, it is also called _retroactively_ for up to 100 most recent messages. This allows you to easily show users their recent messaages and introduce context around the current conversation that might be happening
* It is a fair chunk of code but in pracice, al lwe're doing is taking those new messages` updating our container state, and rendering them in the `MessageList` we defined before.







## Step 9. Sendmessages



## Step 10. Who's online 

## Step 11 Typing indicators

## Step 12. Adding some colour (Styles)









