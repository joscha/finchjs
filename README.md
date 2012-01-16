# Finch.js
Finch.js is a simple, yet powerful, route handling library.  It focusses on easing the assignment of routes, working with parent routes, and handling inline route parameters.


## Basic Usage
### Sounds nifty, where do I start?
To start, you'll need to incldue the Finch.js file on page load

```html
<html>
	<head>
		<title>Using Finch is Fun!</title>
		<script src="./scripts/Finch.js" type="text/javascript" language="javascript"></script>
	</head>
	<body>
		... Stuff ...
	</body>
</html>
```

Once you've included Finch, start setting up the routes that you'd like:

	Finch.route "/home", (params) ->
		console.log("Called home!")

	Finch.route "/home/news", (params) ->
		console.log("Called home/news!")

Lastly, once you're done setting up the routes, start calling them:

	Finch.call "/home"
	Finch.call "/home/news"

Will output this to the console:

	> Called home!
	> Called home/news!

Simple, right?

## Parameters
### Okay, what about inline parameters?
An inline parameter is a parameter that is found within the route (rather than in the query string)

here's some examples of routes with parameters
	
	/home/news/1 -> '1' could be a parameter for a news article id
	/home/news/33/comments -> again, we could get '33'
	/user/stoodder -> here we might want to extract 'stoodder'

For the examples above, we could setup funch to listen for routes like these as follows:

	Finch.route "/home/news/:newsId", (params) ->
		console.log("Looking for news article #{params.newsId}")
	
	Finch.route "/home/news/:newsId/comments", (params) ->
		console.log("Looking for news article #{params.newsId}'s comments")
	
	Finch.route "/user/:username", (params) ->
		console.log("Looking up user #{params.username}")

Calling the example route with the route setup shown above, we'd expect to the the following in the console:
	
	-- Finch.call "/home/news/1"
	> Looking for news article 1

	-- Finch.call "/home/news/33/comments"
	> Looking for news article 33's comments

	-- Finch.call "/user/stoodder"
	> Looking up user stoodder

### What about query string parameters, how do those work?
Query string parameters appear at the and of a uri with the following pattern:

	?key1=val1&key2=val2

Finch handles these similarly to inline parameters.  For exmaple, pretend we had the following route set up:

	Finch.route "/home", (params) ->
		console.log("hello = \"#{params.hello}\"")

... And we called ...

	Finch.call "/home?hello=world"

... We would get the following output

	> hello = "world"

**NOTE** Inline query parameters will always overwrite querystring query parameters

	Finch.route "/home/news/:newsId", (params) ->
		console.log("Called news id #{params.newsId}")
	
	Finch.call "/home/news/33?newsId=666"
	> Called news id 33

## Parent Routes
### What's a parent route?
A parent route is a route that is called before a child route is called.  

Okay, so what does that mean? Let me explain by example:

Imagine that we have a page (/home) with tabs on it for inbox, news, etc.  We could imagine that each of the tabs has a corresponding route (/home/inbox, /home/news, /home/etc).  Whenever we call one of the tabs (child) routes we'd want to run any setup code for the /home route (such as loading in initial user data).  typically, without parent routes, we'd need to do the same setup code in each of the child route calls.  However, with Finch we can easily specify a parent route to call

	Finch.route "/home", (params) ->
		console.log("Setup the home route")
	
	Finch.route "[/home]/news", (params) ->
		console.log("Running /home/news")

	Finch.route "[/home]/inbox", (params) ->
		console.log("Running /home/inbox")

	Finch.route "[/home]/etc", (params) ->
		console.log("Running /home/etc")

Here, the piece of the route wrapped in [] is the parent route.  Running the following on this setup code:

	Finch.call "/home/news"

Would give us:
	
	-- Finch.call "/home/news"
	> Setup the home route
	> Running /home/news

**NOTE** Routes calling parent routes MUST start with the '[', we cannot call routes with parent routes embdedded in the middle of a route:

	Won't work:
	Finch.route "/home/[news]", () -> console.log("ARG!!!!")

### Parent routes are multi level
Pretend now that we wanted to go down another level to get a specific news article

Extending on to our previous examples, we could make a new route:

	Finch.route "[/home/news]/:newsId", (params) ->
		console.log("Looking at news article #{params.newsId}")

Calling the route could give us:

	-- Finch.call "/home/news/33"
	> Setup the home route
	> Running /home/news
	> Looking at news article 33

### Parent routes are cached
Often, we'll be switching routes and we won't need to re-setup our parent's data/structure (in out example we could switch tabs or change news articles). Finch knows this, and will remember what we've called and will ensure that we don't re-call the setup of our routes.  Hence running the following (with our previous parent route examples):

	Finch.call "/home/news/33"
	Finch.call "/home/news/99"

Would give us
	
	-- Finch.call "/home/news/33"
	> Setup the home route
	> Running /home/news
	> Looking at news article 33

	-- Finch.call "/home/news/66"
	> Looking at news article 66

Notice, we didn't run re-execute the /home or /home/news rotues (because we didn't need to)

It is also perfectly acceptable to call only to parent routes, doing something like this:

	Finch.call "/home/news/33"
	Finch.call "/home/news"

Would yield the following:
	
	-- Finch.call "/home/news/33"
	> Setup the home route
	> Running /home/news
	> Looking at news article 33

	-- Finch.call "/home/news"
	> Running /home/news

### And what about using parameters in parent routes?
Again, simple.  We can repeat the pattern above, and extend a new parent route for a news article to get it's comments:

	Finch.route "[/home/news/:newsId]/comments", (params) ->
		console.log("Showing comments for #{params.newsId}")

Calling this would give us (again, still using our examples)

	-- Finch.call "/home/news/33/comments"
	> Setup the home route
	> Running /home/news
	> Looking at news article 33
	> Showing comments for 33

Notice in our setup all we had to do was wrap the parent route in brackets

## Advanced Topics
### Asynchronous routes
Sometimes (most of the time?) you'll want to load a parent route before any of the child route's are executed.  Most likely, you'll want to continue down the route call chain after an ajax request has returned a successful result.  To do this, you'll need to specificy an **Asynchronous Callback**.  To do so is simple.  Just add a second parameter in your callback method for the child callback, like so

	Finch.route "/home", (params, childCallback) ->
		console.log("Calling the home route")
		setTimeout(childCallback, 1000)
	
	Finch.route "[/home]/news", (params) ->
		console.log("Called /home/news")

Doing this will tell Finch that the /home route is asynchronous and will need to wait for the child callback to be executed.  Calling the /home/news route would yield:

	-- Finch.call "/home/news"
	> Calling the home route
	... waits for 1 second
	> Called /home/news

It should also be noted that callback method take in an optional parameters argument.  This allows us to append/change/remove values from the parameters list and pass it along to the child methods (by default, the parameters are always passed down)

For exmaple:
	
	Finch.route "/home", (params, childCallback) ->
		console.log("Calling the home route")
		setTimeout( () ->
			params.hello = 'world'
			childCallback(params)
		, 1000)
	
	Finch.route "[/home]/news", (params) ->
		console.log("Called /home/news, hello = \"#{params.hello}\")

Again, calling the /home/news route would yield:

	-- Finch.call "/home/news"
	> Calling the home route
	... waits for 1 second
	> Called /home/news, hello = "world"

**NOTE** Because of Finch's caching abilities, if a call is interupted (perhaps a use is clicking madly through your website), the current call will be aborted and once finished, will call the newly updated route.

### Setup, Load, and Teardown

	

