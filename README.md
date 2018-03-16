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

Perhaps the best way to learn Chatkit is to start building, so I highly reccomend you follow along. Along the way, you'll best practices when using Chatkit with React.


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








