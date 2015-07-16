# Swagger Client Generator
[![Build Status](https://travis-ci.org/signalfx/swagger-client-generator.svg?branch=master)](https://travis-ci.org/signalfx/swagger-client-generator)

Generates a client given a requests handler and a [Swagger API Specification](https://github.com/wordnik/swagger-spec/blob/master/versions/1.2.md) schema (which can be generated with [fetch-swagger-schema](https://github.com/signalfx/fetch-swagger-schema): `npm install -g fetch-swagger-schema; fetch-swagger-schema <api-docs url> <destination>`).

This is intended to be a helper for creating swagger clients by passing in a request handler for the platform being used (e.g., you XHR request handler in browsers, and request in node). It provides [robust client-side validation](https://github.com/signalfx/swagger-validate) according to the API endpoints and converts given data into appropriate url, header, and body information to be consumed by the request handler.

## Example
```js
function requestHandler(error, request){
  if(error) return console.error(error.toString());

  var xhr = new XMLHttpRequest();
  xhr.open(request.method, request.url)

  if(request.headers){
    Object.keys(request.headers).forEach(function(header){
      xhr.setRequestHeader(header, request.headers[header]);
    });
  }

  xhr.onloadend = function(){
    request.options.callback(this.response);
  };

  xhr.send(request.body);
}

// assumes that 'schema' already exists in scope and is the schema object for
// http://petstore.swagger.io/api/api-docs 
var api = swaggerClientGenerator(schema, requestHandler);

// for apiKey authorization use: api.auth('my-token')
// for basicAuth use: api.auth('username', 'password')
// authorization may be set for any level (api, api.resource, or api.operation)

api.pet.getPetById(2, function(response){
  console.log(response);
});
```
## Creating a schema object
A schema is just a compilation of the Swagger Resource Listing object with each Resource object embedded directly within it. You can fetch one and save it automatically by using [fetch-swagger-schema](https://github.com/signalfx/fetch-swagger-schema):
```shell
# install the fetch-swagger-schema tool
npm install -g fetch-swagger-schema
fetch-swagger-schema <url to a swagger api docs> <destination>
# the generated schema json file will be at <destination>
```

## API
#### `api = swaggerClientGenerator(schemaObject, requestHandler)`
* *schemaObject* - A json object describing the schema (generally generated by [fetch-swagger-schema](https://github.com/signalfx/fetch-swagger-schema)).
* *requestHandler* - A function which accepts two parameters `error` and `request`. The error object will be a [ValidationErrors](https://github.com/signalfx/swagger-validate#swaggervalidateerrorsvalidationerrors) object if an validation error occurs, otherwise it will be undefined. The request object is defined below.
* *api* - An object which can be used as the api for the given schema. The first-level objects are the resources within the schema and the second-level functions are the operations which can be performed on those resources.

#### `api.<resource>`
A map of all the resources as defined in the schema to their operation handler (e.g. `api.pet`).

#### `requestHandlerResponse = api.<resource>.<operationHandler>(data, options)`
This is the actual operation handler invoked by the clients (e.g., `api.pet.getPetById`).

The operation handler takes two parameters:
* *data* - A map of the operation data parameters. If the operation only has one parameter, the value may be used directly (i.e. `api.pet.getPetById(1, callback)` is the same as `api.pet.getPetById({petId: 1}, callback)`).
* *options* - A map of the options to use when calling the operation. This differs based on operation but it may include parameters such as `contentType` or `accept` and it commonly includes options to pass to the request handler (such as a callback function when the request completes). If options is a function, then the `callback` property of options will be a reference to the function for convenience (e.i. `api.pet.getPetById(1, callback)` is the same as `api.pet.getPetById(1, { callback: callback })`.

The operation handler processes the data and options, passes the processed `request` data to the `requestHandler` and returns the result of the requestHandler back to the caller. The operation handler does not process the results of the requestHandler before returning it to the caller. 

#### `url = api.<resource>.<operationHandler>.getUrl(data)`
Intended for advanced use-cases. Returns the URL which would be called if the operation were to be called using this operation handler. The acceptable HTTP method to call this url can be found at `api.<resource>.<operationHandler>.operation.method`.

#### `requestHandler(error, request)`
The request handler is the second parameter for the swaggerClientGenerator and will be called whenever an operation is invoked. If there are any validation errors, the errors parameter be a [ValidationErrors](https://github.com/signalfx/swagger-validate#swaggervalidateerrorsvalidationerrors) object describing the validation errors in detail.

The request object will have the following properties which should be used to issue the actual HTTP request:
* *method* - The HTTP method string for the operation
* *url* - The url string to call for the operation
* *headers* - A map of the request headers
* *body* - The body of the request
* *options* - Options passed in as the second parameter to the operation handler, commonly used for callbacks
* *operation* - The metadata of the operation being invoked
* *data* - The raw data used to generate the url, headers, and body
* *errorTypes* - A map of [all possible error types](https://github.com/signalfx/swagger-validate#swaggervalidateerrorsvalidationerrors) which are used in the error parameter (useful for instanceof checks)

##### `operation` objects
This object, which is found in either `request.operation` or `api.<resource>.<operationHandler>.operation` contains references to the entire schema object:
* [Operation](https://github.com/wordnik/swagger-spec/blob/master/versions/1.2.md#523-operation-object) - `operation`
* [API Object](https://github.com/wordnik/swagger-spec/blob/master/versions/1.2.md#522-api-object) - `operation.apiObject`
* [API Declaration](https://github.com/wordnik/swagger-spec/blob/master/versions/1.2.md#52-api-declaration) - `operation.apiObject.apiDeclaration`
* [Resource Object](https://github.com/wordnik/swagger-spec/blob/master/versions/1.2.md#512-resource-object) - `operation.apiObject.resourceObject`
* [Resource Listing](https://github.com/wordnik/swagger-spec/blob/master/versions/1.2.md#51-resource-listing)- `operation.apiObject.resourceObject.resourceListing`

## Developing
After installing [nodejs](http://nodejs.org) execute the following:

```shell
git clone https://github.com/signalfx/swagger-ajax-client.git
cd swagger-ajax-client
npm install
npm run dev
```
The build engine will test and build everything, start a server hosting the `example` folder on [localhost:3000](http://localhost:3000), and watch for any changes and rebuild when nescessary.

To generate minified files in `dist`:
```shell
npm run dist
```
