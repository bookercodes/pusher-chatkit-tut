
## Step 3. Setup the server

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

const PORT = 3001
app.listen(PORT, err => {
  if (err) {
    console.error(err)
  } else {
    console.log(`Running on port ${PORT}`)
  }
})
```
