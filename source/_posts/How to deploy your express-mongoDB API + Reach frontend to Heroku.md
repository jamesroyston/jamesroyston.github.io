---
title: How to deploy your express-mongoDB API + React frontend to Heroku
date: 2019-07-24 17:58:42
tags:
---

If you're like me, you're probably a frontend dev who enjoys writing JavaScript. But you're also curious about how the "backend" works. If so, follow along as I recap the roadblocks I ran into while trying to implement a task such as the title states. 

**Some assumptions I'm making**
- you already have a react app created.
- you already have an express app created.
- your express app is connected to mongo atlas already.

**Resources I used**

There are a couple of blog posts that go over deploying your react/express app in much higher detail and, quite frankly, they were extremely helpful in my endeavors. The only thing those posts lacked was the mongoDB and mongoAtlas portion. Here's those articles
- [Dave Ceddia](https://daveceddia.com/deploy-react-express-app-heroku/)
- [Chloe Chong](https://medium.com/@chloechong.us/how-to-deploy-a-create-react-app-with-an-express-backend-to-heroku-32decfee6d18)

-----

### Okay, let's get started

1) first copy your react app (the folder containing the project files) is inside of the root of your express project like so:
 ```
 /
|- package.json
|- index.js
|- models/
    |- Posts.js
|- client/
    |- package.json
    |- src/
       |- components/
       |- index.js
       |- app.js
```

2) Now we need to set up the proxy so you can call the express api from React without using `http://localhost:3001` (port number isn't important for this ex). Navigate to your clientside `package.json` file and add:
```json
"proxy": "http://localhost:3001"
```
3) Now we need to replace `http://localhost:3001` with `/api/yourDefaultRoute` in any AJAX calls made in your React app. If you're using Redux, this will likely be in your `actions.js` file(s). If you're using local component state, it'll likely be in any components that use the `componentDidMount()` lifecycle hook to fetch data. Ex: 
```javascript
componentDidMount() {
  fetch('/api/posts')
    .then(res => res.json())
    .then(res => console.log(res))
    .catch(err => console.log(err))
```

4) Now we'll go back into the root directory of your express app and open up `index.js`. We need to make sure node is serving the built version of our clientside app. We also want to ensure we've updated our express routes so that the proxy works.

``` javascript
const cors = require('cors')
const path = require('path')
const Post = require('./models/Post')

// prevents cors headaches when your react app calls your api
app.use(cors())

// serves the built version of your react app
app.use(express.static(path.join(__dirname, 'client/build')))
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname + '/client/build/index.html'))
})

// for your REST calls, append api to the beginning of the path
// ex: 
app.get('/api/posts', async (req, res) => {
  try {
    res.json(await Post.find())
    // Post is a mongoose schema we've defined in /models
    // .find() is a method on the model for fetching all documents
  } catch (err) {
    res.json({message: err})
  }
})

// ensures the proxy we set earlier is pointing to your express api
const port = process.env.PORT || 3001
app.listen(port, () => {
  console.log(`listening on port ${port}`)
});
```

_Here's the 'Post' mongoose schema._

```javascript
const mongoose = require('mongoose')

const PostSchema = mongoose.Schema(
  {
    topic: {type: String, required: true},
    body: {type: String, required: true},
    date: {type: Date, default: Date.now}
  }
)
                      
module.exports = mongoose.model('Post', PostSchema);
```

5) Phew, getting there! Now navigate to your root (express') package.json and add this script: `"heroku-postbuild": "cd client && npm install && npm run build"` to the `"scripts"` object. 

Ok so that concludes the setup in your project folder. Feel free to test that everything still works by running your react app and express api in separate terminals and test your AJAX calls. Everything working? Eff yeah, let's continue!

6) Now we just need to make sure we have heroku installed on our machine, create the heroku app, and run the deploy command. Here's the command for installing heroku. 
```bash
brew tap heroku/brew && brew install heroku
```
(if you are on windows or linux, here's the instructions for those OSes: [https://devcenter.heroku.com/articles/heroku-cli](https://devcenter.heroku.com/articles/heroku-cli))

7) Did that work? Great! Now run each of these, one after the other:
```
git init
heroku create my-project
heroku login //this will redirect you to sign in via your default browser
git push heroku master
```

If all went well, you should see the build logs flood your terminal and the end result should look something like this:

```
-----> Build succeeded!
-----> Discovering process types
       Procfile declares types     -> (none)
       Default types for buildpack -> web
-----> Compressing...
       Done: 49.3M
-----> Launching...
       Released v13
       https://my-project.herokuapp.com/ deployed to Heroku
```
:smile: :fireworks: :fire: :fire: :fire: 

8) LAST and MOST CRUCIAL STEP _IMO_: go into mongo atlas and ensure you've enabled a global IP address whitelist for your mongoDB cluster. If you don't, your app will be running but your data won't ever get fetched. You'll have a network tab full of 503 network errors :sweat_smile: (_this had me stuck for quite a while. Nevermind the fact that I probably should have been asleep at the time I was hammering away on this project at 2am...._)

**SICK, we are all done. Go to your project's URL (provided by the terminal, or via heroku's dashboard on their website) and be amazed by what you've accomplished!**

If you want to see my working example, you can check it out [here.](http://mern-app-msg.herokuapp.com/) :heart:

P.S. This was my first blog post. Feedback is welcome! I hope y'all enjoyed this post and/or found it useful. 

--
James

> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE4ODY4MjY2ODldfQ==
-->