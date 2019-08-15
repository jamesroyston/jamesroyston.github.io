---
title: How to deploy your express-mongoDB API + React frontend to Heroku
date: 2019-07-24 17:58:42
tags: javascript react mongodb heroku nodejs
---

# :wave:

 If you're like me, you're probably a frontend dev who enjoys writing JavaScript but you've never worked with the backend. That said, you probably know, from a birds-eye view, how it _generally_ works. In this article, I'll go over how I used express and mongoDB to write a RESTful api to be used with my React frontend. I'll also go over how to setup a cluster on Mongo Atlas and connect express to it. 

**Assumptions**
- you already have a react app created.
- you already have an express app created.

**Resources I used**

There are a couple of blog posts that go over deploying your react/express app in much higher detail and, quite frankly, they were extremely helpful in my endeavors. The only thing those posts lacked was the mongoDB and mongoAtlas portion. Here's those articles
- [Dave Ceddia's article](https://daveceddia.com/deploy-react-express-app-heroku/)
- [Chloe Chong's article](https://medium.com/@chloechong.us/how-to-deploy-a-create-react-app-with-an-express-backend-to-heroku-32decfee6d18)

--

#### Okay, let's get started

## 1) Combining your clientside and serverside code

First, copy your react app (the folder containing the project files) is inside of the root of your express project so that your file tree looks like this:
 ```
|- package.json
|- server.js
|- models/
    |- Posts.js
|- client/             (the react folder)
    |- package.json
    |- src/
       |- components/
       |- index.js
       |- app.js
```

## 2) Create a mongo atlas account

Navigate to [the mongo atlas site](https://www.mongodb.com/cloud/atlas) and sign up for a free account.

## 3) Setting up the cluster

After you've signed up, we need to configure a mongo atlas project and cluster, and then create our first database on that cluster.

![img](1.png)

- On the next screen you can just click on 'create project' without filling anything out. After that, you'll see the main dashboard. Click on 'build a cluster'.

![img](2.png)

- From here you don't need to mess with any of the options. Simply click on 'create cluster' at the bottom right in the banner. Afterwards you'll see your cluster dashboard:

![img](4.png)

- Click on the connect button from the cluster dashboard and follow the steps for creating a mongo user for the cluster and whitelisting IP addresses. To whitelist all IP addresses (helpful for when we push to heroku), add `0.0.0.0` to the whitelist. 

![img](5.png)

![img](6.png)

- At this point, you can proceed to choose a connection method, select 'connect your application' and copy the string per the instructions on the site.

Note: you'll be replacing the <`password`> portion of that string with the password you created for your cluster's user (you made this like 2 minutes ago lol).

- Quick last thing: from the cluster dashboard, click on collections and select the option to add your own data. From here you can create your first database and collection. I did 'my-db' and 'posts' for the database and collection. 

## 4) Connecting to your cluster from express

Open up `server.js` and add the following code:

```javascript
mongoose.connect(
  process.env.DB_CONNECTION,
  { useNewUrlParser: true },
  () => { console.log('connected to db') }
)

// swap process.env.DB_CONNECTION with your string
```

If you are familiar with the dotenv npm package, you'll have a `.env` file that has a `DB_CONNECTION=mongostring` value. For simplicity, we can just actually use the string instead. 

## 5) Setting up the proxy (clientside)

We need to set up the proxy so you can call the express api from React without using `http://localhost:3001` (port number isn't important for this ex). Navigate to your clientside `package.json` file and add:
```json
"proxy": "http://localhost:3001"
```

We also need to replace `http://localhost:3001` with `/api/yourDefaultRoute` in any AJAX calls made in your React app. If you're using Redux, this will likely be in your `actions.js` file(s). If you're using local component state, it'll likely be in any components that use the `componentDidMount()` lifecycle hook to fetch data. Ex: 
```javascript
componentDidMount() {
  fetch('/api/posts')
    .then(res => res.json())
    .then(res => console.log(res))
    .catch(err => console.log(err))
```

## 6) Setting up the proxy (serverside) 

Go back into the root directory of your express app and open up `server.js`. We need to make sure node is serving the built version of our clientside app. We also want to ensure we've updated our express routes so that the proxy works.

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

_In case you were wondering what `Post` was in that last snippet, here's the 'Post' mongoose schema we're importing._

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

## 7) Add heroku post-build script to serverside package.json

Phew, getting there! Now navigate to your root (express') package.json and add this script: 
```
"heroku-postbuild": "cd client && npm install && npm run build" 
```
to the `"scripts"` object. 

Ok so that concludes the setup in your project folder. Feel free to test that everything still works by running your react app and express api in separate terminals and test your AJAX calls. Everything working? Eff yeah, let's continue!

## 8) Installing and configuring Heroku

We need to make sure we have heroku installed on our machine, create the heroku app via the terminal, and run the deploy command. Here's the command for installing heroku. 
```bash
$ brew tap heroku/brew && brew install heroku
```
(if you are on windows or linux, here's the instructions for those OSes: [https://devcenter.heroku.com/articles/heroku-cli](https://devcenter.heroku.com/articles/heroku-cli))

-- 

Did that work? Great! Now run each of these, one after the other:
```bash
$ git init
$ heroku create my-project
$ heroku login 
# this will redirect you to sign in via your default browser
$ git push heroku master
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

## 9) LAST and MOST CRUCIAL STEP _IMO_: double check that you enabled a global `0.0.0.0` whitelist for your cluster PLS

Go into mongo atlas and ensure you've enabled a global IP address whitelist for your mongoDB cluster (per step 3 in this tutorial). If you don't, your app will be running but your data won't ever get fetched. You'll have a network tab full of 503 network errors :sweat_smile: (_this had me stuck for quite a while. Nevermind the fact that I probably should have been asleep at the time I was hammering away on this project at 2am...._)

## SICK, we are all done. 

Go to your project's URL (provided by the terminal, or via heroku's dashboard on their website) and be amazed by what you've accomplished! Pro-tip: on macOS cmd+click will open links from the terminal in your default browser

If you want to see my working example, you can check it out [here.](http://mern-app-msg.herokuapp.com/) :heart:

P.S. This was my first blog post. Feedback is welcome! I hope y'all enjoyed this post and/or found it useful. 

--
James