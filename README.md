
# Build a Chat App with React and Pusher Chatkit

In this tutorial, you’ll learn how to build a chat app with React and [Chatkit](https://pusher.com/chatkit). 

When we're done, we'll have a chat application complete with **typing indicators**, a **"who's online" list**, and **message history**: 

[Animation of final demo]

If you think this sounds like a lot to tackle in one tutorial, you would normally be right! 

However, because we'll use [Chatkit](pusher.com/chatkit), we can more or less focus exclusively on the front-end React code while Chatkit does the heavy lifting.

## Good to know

Before delving into this walkthrough, it would be good to have a basic understanding of [React](https://reactjs.org/tutorial/tutorial.html)


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

Perhaps the best way to learn Chatkit is to start building so I highly reccomend you follow along. Along the way, you'll learn best practices when using Chatkit with React.

Alright, let's code!

## Step 1. Download the starter template

Rather than start from absoloute scratch, this walkthrough is based on a minimal starter template:

![](https://github.com/bookercodes/pusher-chatkit-tut/blob/master/Screen%20Shot%202018-03-20%20at%2015.12.08.png?raw=true)

As you will see, the starter termplate doesn't contain any interesting logic, just boilerplate we need to run a React application and a simple Node server. 

> **"Sever? No one mentioned a server!"** If you're not too familiar with [Node](https://nodejs.org/en/), don't worry 😊. After the next section, we won't really touch the server.

To get started, download the starter template then run `npm install`:

```
git clone https://github.com/bookercodes/chatkit-react-tutorial chatkit-react-tutorial-template
cd chatkit-react-tutorial-template
npm install
```

(This tutorial assumes the use of `npm`, but the equivalent `yarn` commands will work as well.)

## Step 2. Create a Chatkit instance

Now you've downloaded the starter template, let's create a Chatkit instance.

To create your own Chatkit instance, head to the dashboard, hit **Create new** then give your instance a name. I will call mine “React Chat Tutorial”:

![](https://github.com/bookercodes/pusher-chatkit-tut/blob/master/Screen%20Shot%202018-03-20%20at%2016.22.17.png?raw=true)

In the **Keys** tab, take note of your **Instance Locator** and **Key**. We'll need them both in the next section.


## Step 3. Setup the server

While most interactions will happen on the client, Chatkit also needs a server component to create and manage users securely:

![](https://i.imgur.com/9elZ5SQ.jpg)

We won't authenticate users in this tutorial, but we'll still need to define a route that, when called, creates a Chatkit user.

Start by installing [`pusher-chatkit-server`](https://www.npmjs.com/package/pusher-chatkit-server):

```
npm install --save pusher-chatkit-sever
```

Then update `./server.js`:

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
+        res.sendStatus(200)
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
* Authentication is the action of proving a user is who she says she is. When someone first connects to Chatkit, a request will be sent to `/authenticate` to authenticate her. The server needs to respond with a token (returned by `chatkit.authenticate`) if the request is valid. In our case, we will (naïvely) assume everyone is who they say they are, and return a token from `chatkit.authenticate` no matter what.

Boom 💥! That's all we need to do on the server. Let's move on to the client...

## Step 4. Login 

When someone loads the app, we want to ask them who they are:

Once they hit **Submit**, we'll send their username to the server (to the `/users` route we just defined) and create a a Chatkit user if one doesn't exist. 

To collect the user's name, create a component called `./UsernameForm.js` in in `./src/components/`:

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

Then update `App.js`:

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

Starting from the top,

* First, we import the `UsernameForm` component. It probably looks familiar to you because it uses a common React pattern called controlled components. You can read more about controlled components and React forms [here](https://reactjs.org/docs/forms.html)
* In the `render` function we - you guessed it - render the `UsernameForm` and hook up the `onUsernameSubmitted` event handler
* When `onUsernameSubmitted` is called, we send a POST request to the `/users` route we just defined. If the request is successful, we update `this.state.username` so we can reference it later; otherwise, we `conosle.error` the error


## Step 5. Render ChatScreen

At the moment, we render the `UsernameForm` and it occupies the entire screen:

[Animation highlighting the screen]

Once the username has been submitted, we'll want to transition to a different screen - namely, the chat screen.

To do that, we first need to create a `ChatsScreen.js` component in `./src`:


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

Then update `App.js`:

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
+         currentScreen: 'ChatScreen'
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
+    }
  }
}

export default App
```

## Step 6. Connect to Chatkit

Earlier, we installed `pusher-chatkit-server`. Now we're in client-land, you'll need to install [`pusher-chatkit-client`](https://www.npmjs.com/search?q=pusher-chatkit-client) as well:

```
npm install --save pusher-chatkit-client
```

Then update `ChatScreen.js`:

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

Again, starting from the top:

* First, we import `Chatkit`
* Then instantaite our Chatkit `ChatManager` with our `instanceLocator`, `userId` (from `this.props.userId`), and a custom `TokenProvider`. The `TokenProvider` points to the `/authenticate` route, which we defined earlier
* Once `ChatManager` has been initialised, we can call `connect`. `connect` happens asynchronously and a [`Promise`](https://developers.google.com/web/fundamentals/primers/promises) is returned. If you have followed these steps exaclty, you will connect. That being said, watch out for any `console.error`s in case you you missed something.

## Step 7. Create a Chatkit room

When using Chatkit, all messages are sent to a Chatkit room.

Rooms can be created programatically (on the sever or client using `createRoom`), or in the dashboard Inspector tab.

Creating rooms from the Inspector isn't really a good practice (it's mainly intended for testing) but for the purpose of this walkthrough, we'll do it anyway.

In the dashboard, head to the **Inspector** tab and create a user with any name. I will call mine "Admin":

Then, create a room called "General":

It is really imporatnt to note the unique **room ID** highlighted above. 

Done? Awesome, good job!

## Step 8. Component structure

This step marks a significant point in the walkthrough. 

Now we have our "boilerplate" in place (including a Chakit connection), we can rapidly start to build out chat features. 

Going forward, we'll break down each feature into indepdendant (reusable, if you want!) React components:

[Illustration of the React component structure]

We will create each component as we go along but to make the tutorial a bit easier to follow, let's set out the basic component layout now:

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


Awesome.

## Step 9. Subscribe to messages

I am really excited to show you this!

Now we have a `Chatkit` connection, building chat features become as simple as hooking up Chatkit events to UI components. Here, let me show you.

First, create a stateless `MessageList.js` component in `/src/components`:


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

From the top now,

* Once you connect to Chatkit you get a `currentUser` object that represents the current connected user
* Chatkit is "user-driven" meaning most if not all interactions happen on the `currentUser`
* In this case, we call `subscribeToRoom` on the `currentUser` (`currentUser.subscribeToRoom`)
* `subscribeToRoom` takes an event handler called `onNewMessage` that is called in real-time each time a new message arrives
* Because we specified `100`, `onNewMessage` is also called _retroactively_ for up to 100 most recent messages. This allows you to effortlessly show your user their recent chat history
* There is a fair amount of code here but once you break it down, all we're doing is taking new messages and updating the React state.  The significant chat-related code couldn't be more minimal

## Step 9. Sending messages

We're on a roll!

Next, let's allow users to send messages by first creating a `SendMessageForm.js` component in `./src/components`:

```diff
+ import React, { Component } from 'react'
+ 
+ class SendMessageForm extends Component {
+   constructor(props) {
+     super(props)
+     this.state = {
+       text: '',
+     }
+     this.onSubmit = this.onSubmit.bind(this)
+     this.onChange = this.onChange.bind(this)
+   }
+ 
+   onSubmit(e) {
+     e.preventDefault()
+     this.props.onSubmit(this.state.text)
+     this.setState({ text: '' })
+   }
+ 
+   onChange(e) {
+     this.setState({ text: e.target.value })
+     this.props.onChange()
+   }
+ 
+   render() {
+     const styles = {
+       container: {
+         padding: 20,
+         borderTop: '1px #4C758F solid',
+         marginBottom: 20,
+       },
+       form: {
+         display: 'flex',
+       },
+       input: {
+         color: 'inherit',
+         background: 'none',
+         outline: 'none',
+         border: 'none',
+         flex: 1,
+         fontSize: 16,
+       },
+     }
+     return (
+       <div style={styles.container}>
+         <div>
+           <form onSubmit={this.onSubmit} style={styles.form}>
+             <input
+               type="text"
+               placeholder="Type a message here then hit ENTER"
+               onChange={this.onChange}
+               value={this.state.text}
+               style={styles.input}
+             />
+           </form>
+         </div>
+       </div>
+     )
+   }
+ }
+ 
+ export default SendMessageForm
```

Then  - you guessed it - update `ChatScreen.js`:

```diff
import React, { Component } from 'react'
import Chatkit from 'pusher-chatkit-client'
+ import SendMessageForm from './components/SendMessageForm'
import MessagesList from './components/MessagesList'

class ChatScreen extends Component {
  constructor(props) {
    super(props)
    this.state = {
      currentUser: {},
      currentRoom: {},
      messages: []
    }
+    this.sendMessage = this.sendMessage.bind(this)
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
        return currentUser.subscribeToRoom(
          5599364,
          {
            newMessage: message => {
              this.setState({
                messages: [...this.state.messages, message],
              })
            },
          },
          100
        )
      })
      .then(currentRoom => {
        this.setState({ currentRoom })
      })
      .catch(error => console.error('error', error))
  }
  
+  sendMessage(text) {
+    this.state.currentUser.sendMessage({
+      text,
+      roomId: this.state.currentRoom.id,
+    })
+  }

  render() {
    const styles = {
     ...
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
            <MessagesList
              messages={this.state.messages}
              style={styles.chatList}
            />
+            <SendMessageForm
+              onSubmit={this.sendMessage}
+              onChange={this.sendTypingEvent}
+            />
          </section>
        </div>
      </div>
    )
  }
}

export default ChatScreen
```


* The `SendMessageForm` component is similar to the `WhatIsYourUsernameForm` component we defined earlier (just a standard React form)
* When the form is submitted, we access `currentUser` via `this.state` and call `sendMessage` (remember, most interactions happen on `currentUser`)

Hopefully you can start to see a pattern emerge.

Our `ChatScreen` container manages our appliation state, which we update based on simple Chatkit events. 

## Typing indicators

If you've ever tired to implement your own typing indicators you'll know it can be tricky. In general, more features means more data and more connections to manage. 

With Chatkit, you can add typing indicators with two fundamental lines of code. 

Start by creating a  `TypingIndicator.js` component in `/src/components`:

```diff
+import React, { Component } from 'react'
+
+class TypingIndicator extends Component {
+  render() {
+    if (this.props.usersWhoAreTyping.length > 0) {
+      return (
+        <div>
+          {`${this.props.usersWhoAreTyping
+            .slice(0, 2)
+            .join(' and ')} is typing`}
+        </div>
+      )
+    }
+    return <div />
+  }
+}
+
+export default TypingIndicator
```

Then we'll update `ChatScreen.js`:

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
      currentRoom: {},
      messages: [],
+      usersWhoAreTyping: [],
    }
    this.sendMessage = this.sendMessage.bind(this)
+    this.sendTypingEvent = this.sendTypingEvent.bind(this)
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
        return currentUser.subscribeToRoom(
          5599364,
          {
            newMessage: message => {
              this.setState({
                messages: [...this.state.messages, message],
              })
            },
+            userStartedTyping: user => {
+              this.setState({
+                usersWhoAreTyping: [...this.state.usersWhoAreTyping, user.name],
+              })
+            },
+            userStoppedTyping: user => {
+              this.setState({
+                usersWhoAreTyping: this.state.usersWhoAreTyping.filter(
+                  username => username !== user.name
+                ),
+              })
+            },
          },
          100
        )
      })
      .then(currentRoom => {
        this.setState({ currentRoom })
      })
      .catch(error => console.error('error', error))
  }

  sendMessage(text) {
    this.state.currentUser.sendMessage({
      text,
      roomId: this.state.currentRoom.id,
    })
  }

+  sendTypingEvent() {
+    this.state.currentUser
+      .isTypingIn(this.state.currentRoom.id)
+      .catch(error => console.error('error', error))
+  }

  render() {
    const styles = {
      ...
    }
    return (
      <div style={styles.container}>
        <header style={styles.header}>
          <h2>Chatly</h2>
        </header>
        <div style={styles.chatContainer}>
          <aside style={styles.whosOnlineListContainer}>
            <WhosOnlineList
              currentUser={this.state.currentUser}
              users={this.state.currentRoom.users}
            />
          </aside>
          <section style={styles.chatListContainer}>
            <MessagesList
              messages={this.state.messages}
              style={styles.chatList}
            />
+            <TypingIndicator usersWhoAreTyping={this.state.usersWhoAreTyping} />
            <SendMessageForm
              onSubmit={this.sendMessage}
              onChange={this.sendTypingEvent}
            />
          </section>
        </div>
      </div>
    )
  }
}

export default ChatScreen
```

* Typing indicators come down to two fundamental actions: Calling `currentUser.userIsTyping` when the user is typing (`onChange`) and then listening to `userStartedTyping` and `userStoppedTyping`
* You don't have to tell Chatkit when when someone stops typing by design
* If Chatkit doesn't receive a `userIsTyping` event after a few seconds, it assumes the user has stopped typing. In practice, it's really slick 
* I also want to point out that the `userStartedTyping` and `userStoppedTyping` events are never fired for the user who is typing. If you want to test the typing indicators you'll need to open two windows side by side


## Step 10. Who's online 

Can you feel the momentum? We're on a roll 🙌

To finish up the chat app, let's use Chatkit's "who's online" feature to render a list of users and their real-time online status.

Start by creating a `WhosOnlineList.js` component in `/src/components`:

```diff
import React, { Component } from 'react'

class Item extends Component {
  render() {
    const styles = {
      li: {
        display: 'flex',
        alignItems: 'center',
        marginTop: 5,
        marginBottom: 5,
        paddingTop: 2,
        paddingBottom: 2,
      },
      div: {
        borderRadius: '50%',
        width: 11,
        height: 11,
        marginRight: 10,
      },
    }
    return (
      <li style={styles.li}>
        <div
          style={{
            ...styles.div,
            backgroundColor:
              this.props.presenceState === 'online' ? '#00F469' : '#4c758f',
          }}
        />
        {this.props.children}
      </li>
    )
  }
}

class WhosOnlineList extends Component {
  renderUsers() {
    return (
      <ul>
        {this.props.users.map((user, index) => {
          if (user.id === this.props.currentUser.id) {
            return (
              <Item key={index} presenceState="online">
                {user.name} (You)
              </Item>
            )
          }
          return (
            <Item key={index} presenceState={user.presence.state}>
              {user.name}
            </Item>
          )
        })}
      </ul>
    )
  }

  render() {
    if (this.props.users) {
      return this.renderUsers()
    } else {
      return <p>Loading...</p>
    }
  }
}

export default WhosOnlineList
```

Then - for the last time - update `ChatScreen.js`:


```diff
import SendMessageForm from './components/SendMessageForm'
+import WhosOnlineList from './components/WhosOnlineList'
import MessagesList from './components/MessagesList'
import TypingIndicator from './components/TypingIndicator'

class ChatScreen extends Component {
  constructor(props) {
    super(props)
    this.state = {
      currentUser: {},
      currentRoom: {},
      messages: [],
      usersWhoAreTyping: [],
    }
    this.sendMessage = this.sendMessage.bind(this)
    this.sendTypingEvent = this.sendTypingEvent.bind(this)
  }

  comonentDidMount () {
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
        return currentUser.subscribeToRoom(
          5599364,
          {
            newMessage: message => {
              this.setState({
                messages: [...this.state.messages, message],
              })
            },
            userStartedTyping: user => {
              this.setState({
                usersWhoAreTyping: [...this.state.usersWhoAreTyping, user.name],
              })
            },
            userStoppedTyping: user => {
              this.setState({
                usersWhoAreTyping: this.state.usersWhoAreTyping.filter(
                  username => username !== user.name
                ),
              })
            },
+            userCameOnline: () => this.forceUpdate(),
+            userWentOffline: () => this.forceUpdate(),
+            userJoined: user => {
+              const currentRoom = this.state.currentUser.rooms.find(
+                room => room.id === this.state.currentRoom.id
+              )
+              this.setState({
+                currentRoom,
+              })
+              this.forceUpdate()
+            },
          },
          100
        )
      })
      .then(currentRoom => {
        this.setState({ currentRoom })
      })
      .catch(error => console.error('error', error))
  }

  sendMessage(text) {
    this.state.currentUser.sendMessage({
      text,
      roomId: this.state.currentRoom.id,
    })
  }

  sendTypingEvent() {
    this.state.currentUser
      .isTypingIn(this.state.currentRoom.id)
      .catch(error => console.error('error', error))
  }

  render() {
    const styles = {
      ...
    }
    return (
      <div style={styles.container}>
        <header style={styles.header}>
          <h2>Chatly</h2>
        </header>
        <div style={styles.chatContainer}>
          <aside style={styles.whosOnlineListContainer}>
+            <WhosOnlineList
+              currentUser={this.state.currentUser}
+              users={this.state.currentRoom.users}
+            />
          </aside>
          <section style={styles.chatListContainer}>
            <MessagesList
              messages={this.state.messages}
              style={styles.chatList}
            />
            <TypingIndicator usersWhoAreTyping={this.state.usersWhoAreTyping} />
            <SendMessageForm
              onSubmit={this.sendMessage}
              onChange={this.sendTypingEvent}
            />
          </section>
        </div>
      </div>
    )
  }
}

export default ChatScreen
```

* With Chatkit you can always access a list of users and their online status with `currentRoom.users`
* When users come online (`userCameOnline`) or go offline (`userWentOffline`) we call `forceUpdate` which makes React revaluate `currentRoom.users` and update the UI
* We also need to call `forceUpdate` when new users join (`userJoined`)


Again it really boils down to wiring some simple data and events to React components and that is all, folks!

## Conclusion


















