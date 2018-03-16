
## Step 3. Setup the server

In the the project root, install  `pusher-chatkit-server`:


    npm install --save pusher-chatkit-server

Then, update `server.js`:


    const express = require('express')
    const bodyParser = require('body-parser')
    const Chatkit = require('pusher-chatkit-server')
    const cors = require('cors')
    
    const app = express()
    
    const chatkit = new Chatkit.default({
      instanceLocator: 'v1:us1:542391ba-ff28-4674-a4ad-a464fd59f9f6',
      key:
        'c79c6866-4b06-4848-aca8-56d97a69d3a3:NCebZUHExpE1dhRCYIusaS6yuvYO1au7grCbFuA0oA8=',
    })
    
    app.use(bodyParser.urlencoded({ extended: false }))
    app.use(bodyParser.json())
    app.use(cors())
    
    app.post('/users', (req, res) => {
      const { username } = req.body
      chatkit
        .createUser(username, username)
        .then(() => res.sendStatus(201))
        .catch(error => {
          if (error.error_type === 'services/chatkit/user/user_already_exists') {
            res.sendStatus(200)
          } else {
            res.status(error.statusCode).json(error)
          }
        })
    })
    
    const PORT = 3001
    app.listen(PORT, err => {
      if (err) {
        console.error(err)
      } else {
        console.log(`Running on port ${PORT}`)
      }
    })

It's just a few of lines of code, but there is a lot to unpack here.


- First, we import `Chatkit`
- Then, we instantiate `Chatkit` with the Instance Locator and Key from the dashboard
- Finally, we define the `/users` route which takes the `username` and uses it to create a Chatkit user
