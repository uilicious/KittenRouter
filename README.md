# What is KittenRouter
Kitten Router is a routing script for [Cloudflare Workers](https://www.cloudflare.com/products/cloudflare-workers/) that attempts to connect to a list of specified servers and redirect the request to whichever server that is currently 'alive' at that point of time. It is extremely useful when you have servers that may go down or are unavailable to process the request and KittenRouter can automatically attempt to redirect the request to the next configured URL for processing.

At the same time, it can be configured to log down information to your ElasticSearch server for analytical purposes. Some of the information logged are the status of the servers, country of the request and etc. For the full details, see the `index.js` file.

# How to use KittenRouter
Ultimately, Kitten Router is used together with Cloudflare workers. There are two ways in which you can use Kitten Router on your Cloudflare worker script,

 1. Using NPM modules
 2. Adding Kitten Router manually
 

### 1) Using NPM modules
If you are comfortable in building projects in NodeJS and wanted to deploy it serverlessly on Cloudflare, you can do that as well.

You can use [NPM modules in Cloudflare](https://developers.cloudflare.com/workers/writing-workers/using-npm-modules/), like how we did for Inboxkitten API component, just that you will need webpack to help compact your scripts into a single `.js` file and this single `.js` file will be used in your Cloudflare worker.

Similarly, if you want to use Kitten Router on your Cloudflare script, you can just follow the 7 steps below to achieve it.

#### Step 1: Installing the necessary packages
KittenRouter is published in npm repository and you can install it as an NPM module.
```bash
    # Change to your project directory
    $ cd <to your project>
    # Install kittenrouter
    $ npm install --save @uilicious/kittenrouter
    # Install webpack, we will need it to compact all your files into a single js file
    $ npm install --save webpack
```

#### Step 2: Writing your code
After installing Kitten Router, you can simply create a variable and use it immediately. You can store this script as your entrypoint script such as `index.js`.

```javascript
    const KittenRouter = require("@uilicious/kittenrouter");
    let config = {
        // some configuration that you have for your KittenRouter
    };
    let kittenrouter = new KittenRouter(confg);

    // Usual Cloudflare worker script
    addEventListener('fetch', event => {
        event.respondWith(handleFetchEvent(event))
    })

    // the async function to handle the request
    async function handleFetchEvent(event) {
        // Depends on your logic of how you want to handle the request
        // E.g. only GET requests will need to go through KittenRouter
        let req = event.request;
        if (req.method === "GET") {
            return kittenrouter.handleFetchEvent(event);
        }
        
        // Default behavior
        return await fetch(event);
    }
```

#### Step 3: Setup NPM run commands
At this point, we can create a run command so that we can run it easily. In your `package.json` file, add in
```javascript
{
    // other settings...
    "scripts": {
        // other settings...
        "build-cloudflare": "webpack --mode production index.js"
        // other settings...
    },
    // other settings...
}
```

#### Step 4: Configure target environment in webpack
One thing to note in webpack is that we can specify the environment the Javascript code will be run in. `webworker` is the closest environment to Cloudflare Workers environment. So in your `webpack.config.js`, configure the target to point to `webworker`.

```javascript
module.exports = {
    target: 'webworker'
};
```


#### Step 5 (Optional): Disable minification for your webpack
In your `webpack.config.js`, add in 
```javascript
module.exports = {
    target: 'webworker',
    optimization: {
        // We no not want to minimize our code.
        minimize: false
    }
}
```

#### Step 6: Run NPM command
```bash
    $ npm run build-cloudflare
```

#### Step 7: Deploy to Cloudflare manually
`Step 6` should create a `main.js` inside the `dist` folder, you can then copy the entire `main.js` and paste into your Cloudflare Worker's Editor of your domain.


### 2) Adding Kitten Router manually

#### Step 1: Your own Cloudflare worker script
Given that you have a Cloudflare worker script such as 
```javascript
    // Usual Cloudflare worker script
    addEventListener('fetch', event => {
        event.respondWith(handleFetchEvent(event))
    })
    
    // the async function to handle the request
    async function handleFetchEvent(event) {
        // Default behavior
        return await fetch(event);
    }
```

#### Step 2: Add in the entire KittenRouter script
You can then add the entire `KittenRouter` class, which is basically the `index.js` file in this github project into your script
```javascript
    // methods of KittenRouter ...
    // ...
    class KittenRouter {
        // ...
    }
    
    // Usual Cloudflare worker script
    addEventListener('fetch', event => {
        event.respondWith(handleFetchEvent(event))
    })
    // ...
```

#### Step 3: Add in the configuration
```javascript
    let config = {
        // some configuration that you have for your KittenRouter
    }
    
    // methods of KittenRouter ...
    // ...
    class KittenRouter {
        // ...
    }
    
    // Usual Cloudflare worker script
    addEventListener('fetch', event => {
        event.respondWith(handleFetchEvent(event))
    })
    // ...
```

#### Step 4: Use and deploy
Declare the Kitten Router object and use it in your request process function. You can then deploy this script to your Cloudflare Worker.
```javascript

    let config = {
        // some configuration you have for your KittenRouter
    }
    // methods of KittenRouter ...
    // ...
    class KittenRouter {
        // ...
    }

    // Usual Cloudflare worker script
    addEventListener('fetch', event => {
        event.respondWith(handleFetchEvent(event))
    })

    // Initialize a new KittenRouter object
    let kittenrouter = new KittenRouter(confg);

    // the async function to handle the request
    async function handleFetchEvent(event) {
        // Depends on your logic of how you want to handle the request
        // E.g. only GET requests will need to go through KittenRouter
        let req = event.request;
        if (req.method === "GET") {
            return kittenrouter.handleFetchEvent(event);
        }
        
        // Default behavior
        return await fetch(event);
    }
```

# How to configure KittenRouter
KittenRouter is initialized with a configuration map that contains the routes and log server settings attributes.

### Configuration Map
| Attribute             | Description                                                                               |
|-----------------------|-------------------------------------------------------------------------------------------|
| route                 | An array of servers' url for the KittenRouter to connect to. KittenRouter will attempt to access the urls in the order of the array that was set. |
| log                   | A map that contains the settings for your ElasticSearch server                            |
| disableOriginFallback | Set to true to disable fallback to origin host when all routes fails                      | 

A sample of the configuration map looks like this
```javascript
//
// Routing and logging options
//
module.exports = {

	// logging endpoint to use
	log : [
		{
			// Currently only elasticsearch is supported, scoped here for future alternatives
			// One possible option is google analytics endpoint
			type : "elasticsearch",

			//
			// Elasticsearch index endpoint 
			//
			url : "https://<Your elasticsearch server url>",

			//
			// Authorization header (if needed)
			//
			basicAuthToken : "username:password",

			//
			// Index prefix for storing data, this is before the "YYYY.MM" is attached
			//
			indexPrefix : "test-data-",

			// Enable logging of the full ipv4/6
			//
			// Else it mask (by default) the last digit of IPv4 address
			// or the "network" routing for IPv6
			// see : https://www.haproxy.com/blog/ip-masking-in-haproxy/
			logTrueIP : false,

			// @TODO support
			// Additional cookies to log
			//
			// Be careful not to log "sensitive" cookies, that can compromise security
			// typically this would be seesion keys.
			// cookies : ["__cfduid", "_ga", "_gid", "account_id"]
		}
	],

	// Routing rules to evaluate, starting from 0 index
	// these routes will always be processed in sequence
	route : [
		// Lets load all requests to commonshost first
		"commonshost.inboxkitten.com"

		// If it fails, we fallback to firebase
		//"firebase.inboxkitten.com"
	],

	// Set to true to disable fallback to origin host 
	// when all routes fails
	disableOriginFallback : false,
}
```