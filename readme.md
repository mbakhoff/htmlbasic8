# Integrating web services

In this tutorial we're going to integrate our forum app with [Tumblr](https://www.tumblr.com/), a sharing site.
This is the functionality we're going to add:
* the user can link their Tumblr account to their forum account from the preferences page,
* when the user adds a post that contains `!tumble`, then the post is also posted on the user's Tumblr blog,
* when the user's post contains `!images:<tumblr-permalink>`, then the images in the linked Tumblr post will be displayed under the user's post.

## Web services overview

Many larger web apps have a public API that lets you use their functionality.
Most APIs use the an architectural style called *REST*.
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
For example, to get only the threads of a specific user, the user id could be passed as a parameter: `GET /api/threads?user=mbakhoff`.
In case of `POST` requests, the parameters are sent in the request body using the `application/x-www-form-urlencoded` encoding (same as HTML forms).

## API security

Most web services won't accept requests from any anonymous user and require some kind of authentication.
User oriented web services tend to differentiate between two categories of requests:
* requests for public resources
* requests for a particular user's resources

To use a public resources, most web services have you register your application and give you an *API consumer key* and a *API consumer secret* (strings).
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

The entire process is described in detail on the [OAuth Core 1.0A specification](https://oauth.net/core/1.0a/).

OAuth 2.0 is very similar, with some differences:
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

TODO OAuth + `!tumble`
