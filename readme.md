# Integrating web services

In this tutorial we're going to integrate our forum app with [Tumblr](https://www.tumblr.com/), a sharing site.
This is the functionality we're going to add:
* the user can link their Tumblr account to their forum account from the preferences page,
* when the user adds a post that contains `!tumble`, then the post is also posted on the user's Tumblr blog,
* when the user's post contains `!images:<tumblr-permalink>`, then the images in the linked Tumblr post will be displayed under the user's post.

## URL syntax and encoding

Before we begin, it's time to take a deeper look at URL syntax and encoding.
The official syntax is:
```
scheme:[//[user[:password]@]host[:port]][/path][?query][#fragment]
```

On the server side we really only care about some parts of it:
```
scheme://host[:port][/path][?query]
```

* *scheme* should always be *https* for web sites (or *http* in case of an insecure site).
  specifies which protocol to use when communicating with the server.
* *host* is the dns name of the server.
  it's used with the port to create the actual network connection to the server.
* *port* is only specificed when the server is not listening for connections on a standard port.
  *https* uses port 443 by default, while *http* uses port 80.
* *path* identifies a resource on the server.
  it's usually either a file name or a servlet mapping in case of java servers.
* *query* is an optional string that can be used to pass parameters to the server.
  **the query string is placed after the path and its start is marked with a question mark `?`**.
  the query string is usually contains a list of key-value pairs.
  a key and a value are separated by the equals sign `=` and key-value pairs are separated by an ampersand `&`.

An example of an URL: `https://example.com/summary?id=3&name=mbakhoff`:
* scheme is `https`
* host is `example.com`
* port is the default 443 for https
* path is `/summary`
* query string is `id=3&name=mbakhoff` and it contains the two key-value pairs

Encoding an URL is quite a mess.
The encoding is important when the path needs to contain special symbols, including but not limited to:
* the path needs to contain a `?` symbol that doesn't mark the start of the query
* a key or value in the query string needs to contain a `=` or `&` symbol that's not a separator

Don't forget that the *path* and *query* are different things.
In fact, **they are encoded using different rules**.
When building an url, always use the right methods for encoding different parts of the URL:
* encode *path segments* using `org.springframework.web.util.UriUtils#encodePathSegment`
* encode *query string* keys and values using `org.springframework.web.util.UriUtils#encodeQueryParam`

This is how a bullet-proof solution might look like:
```java
String hope = "correct encoding? grade = A+ & success";
String url = "https://localhost:8443"
               + "/sample/" + UriUtils.encodePathSegment(hope, "UTF-8")
               + "?sample=" + UriUtils.encodeQueryParam(hope, "UTF-8");
// result is (with added line breaks for easy comparison):
// http://localhost:8080
//   /sample/correct%20encoding%3F%20grade%20=%20A+%20&%20success
//   ?sample=correct%20encoding?%20grade%20%3D%20A%2B%20%26%20success
```
The special characters are encoded using [Percent-encoding](https://en.wikipedia.org/wiki/Percent-encoding).
However, the special characters are different for the path part and the query part.

See this excellent article on how not to encode URLs:
[What every web developer must know about URL encoding](https://www.talisman.org/~erlkonig/misc/lunatech%5Ewhat-every-webdev-must-know-about-url-encoding/).

## Web services overview

Many larger web apps have a public API that lets you use their functionality.
Most APIs use an architectural style called *REST*.
In *REST*, the API exposes different resources of the web app (such as users or forum threads) on corresponding URLs.
The API consumer can access and change the resources using regular HTTP methods: *GET*, *POST*, *PUT* and *DELETE*.
Most modern APIs return data in the *JSON* format in the response.

The HTTP methods are used as they were originally described in the [HTTP specification](https://tools.ietf.org/html/rfc2616#section-9):
* *GET* for retrieving resources
* *PUT* for changing existing resources
* *POST* for creating new resources
* *DELETE* for deleting existing resources

For example, our forum app's *REST* API would look something like this:
* `GET /api/threads` would return info on all the threads
* `GET /api/threads/1` would return info on the thread with id 1
* `POST /api/threads` would create a new forum thread
* `POST /api/threads/1/posts` would create a new post in the thread with id 1
* `PUT /api/threads/1` would modify an existing thread with id 1
* `DELETE /api/threads/1` would delete the thread with id 1

A standard practice is to allow additional parameters to the API using request parameters.
For example, to get only the threads of a specific user, the user could be passed as a parameter: `GET /api/threads?user=mbakhoff`.
In case of `POST` requests, the parameters are sent in the request body using the `application/x-www-form-urlencoded` encoding (same as HTML forms).

## API security

Most web services won't accept requests from any anonymous user and require some kind of authentication.
User oriented web services tend to differentiate between two categories of requests:
* requests for public resources
* requests for a particular user's resources

To use a public resource, most web services have you register your application and give you an *API consumer key* and a *API consumer secret* (strings).
Your applicaion will include the *consumer key* as a request parameter when making requests to the API.
This way the service will know who is making the request.

Using some user's private resources (posting a forum post as some user, changing the user's settings etc) on another web service is more complicated.
The web service won't let your application act on behalf of its user unless the user gives your application explicit permission to do so.
For example, you can't make posts as a arbitary Tumblr user even if you register your application in Tumblr.
The Tumblr user must first tell Tumblr that your application is allowed to act on his/her behalf.

There exists a standard protocol for getting permissions to act on behalf of some user in another web app.
The protocol is called **OAuth** and different versions of it are supported by Tumblr, Google, Facebook, Twitter etc.

This is how OAuth 1.0a works:
* The user visits your web app.
  Your web app needs to use the API of another service.
* Your app sends a request to the other service and includes your *consumer key* and a *redirect-url*.
  The other service responds with an unauthorized *request token*.
* Your web app redirects the user to the other service.
  Your application's adds the unauthorized *request token* as request parameter.
* The other service identifies your app using the *request token*.
  The other service asks the user if he/she wants to give your app permissions.
  If the user grants the permissions, then the other service will mark the *request token* as authorized and redirect the user back to your app (to the *request-url* you passed earlier).
  The other service adds the (now authorized) *request token* and a *verification code* as the request parameters.
* Your application receives the *request token* and the *verification code*.
  Your application sends a request to the other service and includes its *consumer key*, the *request token* and the *verification code* as the request parameters.
  The other service will respond with an *access token* and an *access token secret*.
  The *access token* can be used to make requests on behalf of the user.
* All requests are digitally signed using your *consumer secret*.

The entire process is described in detail in the [OAuth Core 1.0A specification](https://oauth.net/core/1.0a/).

OAuth 2.0 is very similar, with some differences that make it easier to use:
* The step for getting the unauthorized request token is skipped.
* Your application can specify which permissions it needs for the user (OAuth 1.0 authorizes all or nothing).
* The *verification code* is not used.
* The requests are not signed and HTTPS is mandatory.

## Linking to Tumblr posts

We will start with the `!images:<tumblr-permalink>` functionality.
This only uses public resources, so we don't need OAuth (yet).

Create a Tumblr user if you don't have one.
[Register a new application](https://www.tumblr.com/oauth/apps) to get a consumer key:
* application website can be anything, for example github.com/yourname
* set the default callback url to `https://localhost:8443/default_callback`.
  if all goes well, this url is never used.

Tumblr will give you a consumer key and a consumer secret.
Add these to your application.properties:
```text
tumblr.consumer.key=your key
tumblr.consumer.secret=your secret
```

### Parsing permalinks

The first step is to process all user posts and pick out Tumblr permalinks.
You can get a permalink from the share button of a Tumblr post.
A permalink looks like this:
```
http://theartofanimation.tumblr.com/post/164978051225/niykoubou-httpstwittercomniykoubou
```
The permalink contains the *blog-identifier*, e.g. `theartofanimation.tumblr.com` and the *post id*, e.g. `164978051225`.

Create a new class `TumblrLinkProcessor`:
```java
@Component
public class TumblrLinkProcessor {

  @Autowired
  public TumblrLinkProcessor(Environment env) {
    // env contains all properties from application.properties
    String consumerKey = env.getProperty("tumblr.consumer.key");
  }

  public void process(ForumPost post) {
    // TODO: implement
    // replace ForumPost with whatever class you use to store forum posts
  }
}
```

Find the request handler in your forum code that accepts new forums posts.
Use Spring's dependency injection to get an instance of the `TumblrLinkProcessor`: add a `TumblrLinkProcessor` parameter to the controller's constructor and mark the constructor with `@Autowired`.
Let the `TumblrLinkProcessor` process each post before saving it into a database.

The `process` method in the `TumblrLinkProcessor` class takes a forum post as a parameter.
The method should search the forum post's text for permalinks (`!images:<permalink>`).
All the permalinks in a post should be removed and collected for processing.

Example, given the input post:
```text
I like pictures of Art of Animation!
!images:http://theartofanimation.tumblr.com/post/164978051225/niykoubou-httpstwittercomniykoubou
And of Mingjue!
!images:http://jigokuen.tumblr.com/post/134846032190/silk-4-cover
```

The post should be changed to have the text:
```text
I like pictures of Art of Animation!

And of Mingjue!

```
The following permalinks should be collected:
* http://theartofanimation.tumblr.com/post/164978051225/niykoubou-httpstwittercomniykoubou
* http://jigokuen.tumblr.com/post/134846032190/silk-4-cover

### Fetching the Tumblr image links

The permalinks we collected only point to a Tumblr post, but not the images it contains.
To get the actual images, we must ask Tumblr for details.

Open the [Tumblr API docs](https://www.tumblr.com/docs/en/api/v2) and find the URL for reading a published post.
The request needs three parameters: *api_key*, *blog-identifier* and the *id*.
We already have the api key and the permalink contains the other values.
Send a request to Tumblr for each permalink that was collected.

You can send a HTTP request using the [HttpClient](https://hc.apache.org/httpcomponents-client-ga/) library:
```java
URI requestUri = new URIBuilder("https://<url here>")
    .addParameter(parameterName, parameterValue)
    .addParameter(parameterName, parameterValue)
    .build();

try (CloseableHttpClient http = HttpClients.createDefault();
     CloseableHttpResponse response = http.execute(new HttpGet(requestUri))) {
  InputStream responseBody = response.getEntity().getContent();
  // read the response
}
```

Note that you can send the same request from the browser to test the request URL.
The `addParameter` method adds a regular request parameter to the URL (works like `application/x-www-form-urlencoded` used in html forms).
Sample of an URL with query parameters: `https://some.url/path?param1=value1&param2=value2&param3=value3`.

The response body contains a JSON object.
You can parse the JSON object using the [Jackson2](https://github.com/FasterXML/jackson-databind) library.
The easiest way is to use the tree model.

The tree model let's you pick out values from a complex JSON structure by specifying a path to the value.
Given the input JSON:
```json
{
   "meta": {
      "status": 200,
      "msg": "OK"
   },
   "response": {
      "total_users": 2684,
      "users":  [
        {
           "name": "david",
           "following": true,
           "url": "http:\/\/www.davidslog.com",
           "updated": 1308781073
        },
        {
           "name": "ben",
           "following": true,
           "url": "http:\/\/bengold.tv",
           "updated": 1308841333
        }
     ]
   }
}
```

You can read the JSON using the tree model:
```java
JsonNode root = new ObjectMapper().readTree(json);
System.out.println("status: " + root.at("/meta/status").asInt());
System.out.println("total users: " + root.at("/response/total_users").asInt());
for (JsonNode user : root.at("/response/users")) {
  System.out.println("user name: " + user.at("/name").asText());
}
```

Analyze Tumblr's response for each permalink.
The response contains the type of the post.
If the type is not 'photo', then the permalink and the response can be ignored.
Otherwise, collect the photo urls from the response (one post can contain multiple photos).

Change the forum post class so it can store the photo urls in the database.
Store all the photo urls from the permalinks in the forum post.
Change the forum thread template to show all the photos under the post that contained the `!images` directive.

## Adding Tumblr user integration

In this part we're going to add the feature of sharing a forum post directly on the user's Tumblr blog.
These are the steps:
* get access to the user's Tumblr account using OAuth
* identify the user's blog
* add a new forum post processor for the `!tumble` keyword, which uploads the post to Tumblr

### Linking the Tumblr account

To get access to the user's account, we will have to implement the OAuth 1.0A workflow:
* get an unauthorized *request token* from Tumblr
* redirect the user to Tumblr to confirm access
* accept the *verification code* from Tumblr when the user is redirected back to our server from Tumblr
* exchange the *reqest token* and *verification code* for an *access token*
* store the *access token* for later use (used to access user's account via Tumblr API)

Create a new request handler on the server that starts the OAuth workflow.
Add a new button to the preferences page which makes a *POST* request to the handler.
The request handler doesn't take any parameters.
The request handler will send a *POST* request to Tumblr's *request token endpoint* (*https://www.tumblr.com/oauth/request_token*) and include the *consumer key* and a *callback url*.
The *callback url* is the address where Tumblr will redirect the user after he/she has granted access to our app.
Create a new request handler on the server and use its address as the *callback url* (e.g. *https://localhost:8443/something/something*).

According to the OAuth 1.0A specification, the request must also contain a timestamp, nonce, version, signature method and a signature of the entire request.
This is quite a mess and we don't want to do any of these things manually.
Instead, we will use a library called *ScribeJava*, which will automate all of it (OAuth2 is more simple, the signatures are not used).

This is how you use *ScribeJava*:
```java
OAuth10aService service = new ServiceBuilder(consumerKey)
    .apiSecret(consumerSecret)
    .callback(callbackUrl)
    .build(TumblrApi.instance());

OAuth1RequestToken requestToken = service.getRequestToken();
```

You must specify the *consumer key*, *consumer secret* and the *callback url* to the `ServiceBuilder`.
The first two are already in *application.properties*.
It's recommended to also add the *callback url* to *application.properties*.
When you start the forum app on a public server, then it's address is not localhost and the server admin can update the *callback url* in the properties file without recompiling the forum code.
Recall that you can dependency inject the `Environment` object for accessing the properties.

The `getRequestToken()` method will send a request to Tumblr and return a new unauthorized *request token*.
As with *consumer key*, the *request token* has a companion *request token secret*.
Both are contained in the `OAuth1RequestToken` object.
You must store these values for later usage (use a cookie, the database or store them directly in the controller).

The next step is to redirect the user to Tumblr, so that he/she can grant us access.
The url is *https://www.tumblr.com/oauth/authorize*, with a request parameter named `oauth_token` containing the *request token*.
You can conveniently generate the url using *ScribeJava*: `service.getAuthorizationUrl(requestToken)`.

When the user has granted access to our application, Tumblr will redirect them back to our *callback url*.
Our server should receive a *GET* request from the user, containing two request parameters: `oauth_token` and `oauth_verifier`.
Use `@RequestParam` or `HttpServletRequest#getParameter` to access the parameters.

The `oauth_token` is the same *request token* value that we stored earlier.
To finish the OAuth workflow, we need the `OAuth1RequestToken` object again (same that we created in the beginning of the workflow).
If you stored the entire object somehow, then use that.
If you only stored the *request token* and *request token secret* values, then create a `OAuth1RequestToken` from the stored values.

We should now have the `OAuth1RequestToken` and the `oauth_verifier` value.
Send them both to Tumblr using a *POST* request to *https://www.tumblr.com/oauth/access_token*.
Again, the request needs to contain the signature, nonce, timestamp, etc.
You can use *ScribeJava* to automate all of it: `service.getAccessToken(requestToken, verifier)`.

The `getAccessToken` method will send a request to Tumblr and return an *access token*.
This is the reward for the entire long and complicated OAuth workflow.
As with other tokens, the *access token* consists of the token and the associated secret.
Store both the values in the database for later use.

Note: consider that multiple users could be enabling Tumblr integration at the same time.
This means you'll have multiple request tokens and redirects going on in parallel.
Make sure the data structures are thread safe and the tokens are not mixed up.

### Identifying user info

Now that we have the *access token*, it's rather easy to access the user's Tumblr account.
The Tumblr API docs describe different methods that you can use.
Each API method that uses OAuth authentication must be signed with the access token that we stored earlier.
This can be done using *ScribeJava*:
```java
OAuthRequest request = new OAuthRequest(Verb.GET/POST, "https://some api url");
service.signRequest(accessToken, request);
String responseBody = service.execute(request).getBody();
```

Send a request to Tumblr and get the user's information (you can do it right after receiving the access token in the callback handler).
The result should contain a JSON object with a list of the user's blogs.
Use Jackson2 as earlier to get the blog urls from the response JSON.

Pick the first blog from the list and store it in the database.
This is where we'll add the forum posts with `!tumble`.

### Processing `!tumble` directives

Create a new class `TumbleProcessor` with a method `public void process(ForumPost post)`.
The processor should look for the `!tumble` keyword.
If it exists, remove it and create a new Tumblr blog post.

Dependency inject the processor into the controller that handles adding new posts.
Run the new `process` method on all new posts.
Make sure that `TumbleProcessor` is annotated with `@Component`, otherwise Spring can't find it and inject it.

Use Tumblr's API to add the blog post.
The post type should be text.
You'll need to add a parameter named *body* to the request.
Use `OAuthRequest#addParameter` to add parameters.
Make sure to sign the request with the user's access token (add parameters before signing).

Tumblr can take some time to process your API request (2-3 seconds).
We want our forum to be blazing fast and such delays are unacceptable.
`TumbleProcessor` should not block adding the forum post while Tumblr is doing its thing.
Create a thread (or use an [executor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executors.html)) in the `process` method and start the Tumblr request in the background.
