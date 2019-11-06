# api-wrapper-template Usage

Once you've used this tool to scaffold a project, you still need to write the actual code to wrap the API in your CFML application. The following is a brief guide to the conventions and structure of the API wrapper component generated by this project.

*Note: I use the terms API wrapper, API client, API library, and REST SDK interchangeably.* 

The name of the API you are wrapping is used to generate a `cfc` file, which is the core of the API wrapper component that you're writing. Within that file, you'll need to make some manual adjustments, to ensure the wrapper is configured properly to work with the API. These adjustments primarily have to do with Authentication.

## Authentication

Making sure that authentication is handled correctly is the first step in building your API client. Obviously, if the API has no authentication, you can skip this section.

### Basic Authentication

**tldr;** *Handled in the `cfhttp` call, using the `username` and `password` variables.*

If you've scaffolded your wrapper with the `basic` option, the assumption is that the API requires a `username` and `password`. In actuality, this is not how most APIs function, so you make need to make some changes. By default, the `init()` method for a wrapper using Basic Authentication is structured like this:

```cfc
public any function init(
  string username = '',
  string password = '',
  string baseUrl = 'https://api.example.com/v1',
  boolean includeRaw = false ) {...}
```

The `username` and `password` are then saved in the variables scope, and used for authentication in the `cfhttp` call in the `apiCall()` method:

```cfc
cfhttp( url = fullPath, method = httpMethod, username = variables.username, password = variables.password, result = 'result' ){...}
```

Depending on how the API that you are working with handled Basic Authentication, you should modify these two portions of the wrapper to more effectively handle it.

### API Key Authentication

**tldr;** *Set in the `getBaseHttpHeaders()` method, using the `apiKey` variable, which is then provided as a header.*

For wrappers scaffolded with the `apikey` authentication option, you'll need to modify the wrapper. This is because API providers implement API keys in a range of ways, so there's no simple way to scaffold a wrapper to handle it. By default, the `init()` method for a wrapper using API keys for authentication is structured like this:

```cfc
public any function init(
  string apiKey = '',
  string baseUrl = 'https://api.example.com/v1',
  boolean includeRaw = false ) {...}
```

That API key is then passed as a header in every request to the API, because it is included in the `getBaseHttpHeaders()` method (you can learn a little more about this method [below](#getbasehttpheaders)):

```cfc
private struct function getBaseHttpHeaders() {
  return {
    'Accept' : 'application/json',
    'Content-Type' : 'application/json',
    'Authorization' : 'Bearer #variables.apiKey#',
    'User-Agent' : 'example/#variables._example_version# (CFML)'
  };
}
```

You will need to adjust the `init()` method and this base HTTP header struct in order to account for the way any particular API handles API keys. That is, if the API requires an `applicationId` and `applicationKey`, you would update the `init()` method to accept them, and then update the `getBaseHttpHeaders()` method to include them, named according to the API's requirements.

If an API doesn't use headers for authentication, you'd need to make more substantial changes in order to include the API key in the URL parameters or request body.

### Environment Variables

In order to separate code from config, API wrappers scaffolded with this project are structured to accept authentication credentials as environment variables. This is done via the `secrets` struct in the `init` method. In API clients where you need to provide an API key, it will look like this:

```cfc
var secrets = {
  'apiKey': 'EXAMPLE_API_KEY'
};
```

The value on the right is the name of an environment variable. If it is found, the key on the left will be set equal to its value. So, in the example about, if there were an environment variable named `EXAMPLE_API_KEY` with a value of `abc123!`, when the wrapper is instantiated in your application, it would set `variables.apiKey` equal to `abc123!`, and you would be able to use that for authentication.

If you provide credentials when you initialize the wrapper component, they take precedence over any environment variables.

**The names of the credentials and environment variables should be updated to suit your needs and the requirements of the API.**

## API Endpoints and Method Structure

**tldr;** *API endpoints are mapped, one-to-one, to functions within the core wrapper component, and all HTTP requests are delegated to a single `apiCall()` method.*

When you wrapper is initialized, it saves the API's base URL (https://api.example.com) and any supplied authentication credentials in its `variables` scope. Actual requests to the API are then handled via functions within the component, which you will need to write. These functions are mapped, one-to-one, to the API's endpoints. In this way, the component and its functions "wrap" the functionality of the API.

Each public method within your wrapper should 1) accept the parameters required for interacting with the corresponding endpoint, 2) ensure they are assembled correctly, and 3) then pass them on to the component's `apiCall()` method, which does the actual work of handling the HTTP request. The API's response is then automatically returned to the caller as a struct.

Here is the basic template for your wrapper's functions:

```cfc
public struct function methodName( required any argument ) {
  return apiCall( HTTP_METHOD, API_ENDPOINT, PARAMS, BODY, HEADERS );
}
```

So, as an example, a function to retrieve an image from some API might look like this:

```cfc
public struct function getImage( required numeric id ) {
  var params[ 'imageId' ] = id;
  return apiCall( 'GET', '/images/image', params );
}
```

For a better understanding of how the `apiCall()` method works, look at its [documentation](#apicall-required-string-httpmethod-required-string-path-struct-queryparams----any-payload---struct-headers) below.

Once you've handled authentication, the process of writing your API wrapper primarily consists of writing these functions for each endpoint of the API.

## Important Methods
When you're writing an API wrapper using this tool, it's helpful to understand the following methods and their purpose. While the wrapper does have additional private methods, they primarily exist in order to assist the methods listed below.

#### `init()`

As discussed above, the primary purpose of this method is to set constants that will be used by the wrapper. These include the base endpoint for the API (https://api.example.com), and any credentials required for authentication. It also comes with an `includeRaw` option, which can be used to return additional information about the requests your wrapper is making to the API in the response struct.

#### `apiCall( required string httpMethod, required string path, struct queryParams = { }, any payload = '', struct headers = { } )`

The hard work of interacting with the API is handled by this method. All your wrapper's public methods, which interact with the API's endpoints, will delegate to this function. It takes the elements of an HTTP request, assembles them, and then passes them on to [`makeHttpRequest()`](#makehttprequest) in order to make the HTTP request to the API. It then processes the response and returns it to the caller. 

Assuming the `Content-Type` of the request is `application/json`, if the `payload` is a ColdFusion struct or array, it is automatically serialized. If it is JSON already, it is simply passed to the API. If the payload is being sent with a `Content-Type` of `application/x-www-form-urlencoded` or `multipart/form-data`, the keys/values of the payload struct are automatically sent as form fields.

The struct of `queryParams` will automatically be assembled and added to the URL.

#### `getBaseHttpHeaders()`

This returns a struct of keys/values that are passed as headers in every request to the API. By default it includes the standard headers: `Accept`, `Content-Type`, and `User-Agent`.

The `Content-Type` and `Accept` here default to `application/json`, but you may need to change them, depending on the API you are working with. 

This method is typically where you define authentication headers required to work with the API.

#### `makeHttpRequest()`

This method handles the actual `cfhttp` request. The response is passed back to the `apiCall()` method for handling, before it's returned to the client.