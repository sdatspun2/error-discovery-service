# Error Catalog Service for HTTP APIs

Error Catalog Service provides operations to manage error types and error catalogs associated with HTTP APIs. Such a service could provide a dictionary of errors for HTTP APIs. This document provides a rationale behind such a service and describes the API of the service.

This API is licensed under [The Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0.html).

## Terms Used

#### Resource

The key abstraction of information in REST is a resource. A [resource](https://github.com/paypal/api-standards/blob/master/api-style-guide.md#resource) is a conceptual mapping to a set of entities, not the entity that corresponds to the mapping at any particular point in time. More precisely, a resource R is a temporally varying membership function MR(t), that for time t maps to a set of entities, or values, that are equivalent. The values in the set may be resource representations and/or resource identifiers.

#### Service

Service provides a generic Application Programming Interface (API) for accessing and manipulating the value set of a resource, regardless of how the membership function is defined or the type of software that is handling the request.

#### Namespace

Capabilities drive service modeling and namespace concerns in an API portfolio. Namespaces are part of the Business Capability Model. Examples of namespace are: identity, compliance, settlements, transfers, credit, payments, error, etc.

#### Problem Detail

[RFC 7807](https://tools.ietf.org/html/rfc7807) defines "problem detail" as a way to carry machine-readable details of errors in a HTTP response to avoid the need to define new error response formats for HTTP APIs.

## Rationale

This section lists reasons to use the Error Catalog Service for HTTP APIs.

### Increased support cost

It is often observed that there is non-trivial and hidden cost of supporting HTTP API consumers to navigate error scenarios. There could be various reasons behind this cost including but not limited to the following.

* Errors are not documented with API contracts, documentation is not comprehensive, error description is often confusing or incorrect.
* Errors do not provide enough context of the problem for forensic and diagnosis of the problem.
* API documentation is out of sync with the implementation.
* Not enough testing is done for error scenarios to validate if API operations exposed by the services are emitting errors as documented.

### Out of sync error documentation

API consumers should be able to refer with confidence an API's documentation for errors generated by various operation(s). This helps to reduce the cost and the time spent supporting an API and increases adoptability of the API. However, there aren't many tools available to help the API developers keep their API's error documentation and service implementation in sync.


### Sending errors with problem details is not sufficient

There are many [implementations](http://apistylebook.com/design/topics/http-status-standard-error) of error schema used for sending errors in HTTP response. Most notably, [Problem Details for HTTP APIs, RFC 7807](https://tools.ietf.org/html/rfc7807) defines a "problem detail" as a way to carry machine-readable details of errors in a HTTP response. It provides a client enough information in machine-readable format so that the client can treat it appropriately.

Also, the RFC 7807 helps in avoiding the need to define new error response formats for HTTP APIs. It provides a data model for problem details in `JSON` [Section 3](https://tools.ietf.org/html/rfc7807#section-3) and `XML`. The [Problem Details Object](https://tools.ietf.org/html/rfc7807#section-3.1) includes the following members that are relevant for our discussion.

* `type` A URI reference [RFC3986](https://tools.ietf.org/html/rfc3986) that identifies the problem type. RFC 7807 specification encourages that, when dereferenced, it provide human-readable documentation for the problem type (e.g., using HTML [W3C.REC-html5-20141028](https://tools.ietf.org/html/rfc7807#ref-W3C.REC-html5-20141028)).
* `title` A short, human-readable summary of the problem type.  It SHOULD NOT change from occurrence to occurrence of the problem, except for purposes of localization (e.g., using proactive content negotiation; see RFC7231, [Section 3.4](https://tools.ietf.org/html/rfc7231#section-3.4)).
* `detail` A human-readable explanation specific to an occurrence of the problem. Often this would be a template string where variables are filled based on the context of an occurrence of a problem. For example, "Payment transaction amount {} exceeds account balance {}." is a template string. Here, `{}` represents variables that could be filled up based on a specific occurrence of the problem.

#### Questions

Following important questions regarding "problem details" still need to be addressed while implementing HTTP APIs. Note that even if an HTTP API is not using error schema suggested by RFC 7807, these questions may still be relevant.  

1. How does an API developer get values for `type`, `title` and `detail` related to a problem in the service implementation? Hard coding these in the service code would not be advisable from both maintenance and localization reasons.
2. How does the API developer publish problem types with relevant API operations so the consumers of the API can program client applications to handle these appropriately? Remember, OpenAPI does not have any construct to describe problem types associated with operations.
3. How can the API developer keep problem `type`, `title`, `detail` and other metadata consistent between the service implementation and the API documentation?
4. How can API developer retrieve localized strings for `title` and `detail` of a problem type?

### OpenAPI specification does not support problem types

Widely used [OpenAPI Specification](https://swagger.io/specification/) (OAS) defines a standard, language-agnostic interface for RESTful APIs which allows both humans and computers to discover and understand the capabilities of the service without access to source code, documentation, or through network traffic inspection. A consumer can understand and interact with the remote service with a minimal amount of implementation logic.

For each operation in OpenAPI document, one can define possible HTTP status codes and the response body schema. However, the specification does not provide any component schema element(s) (fields, objects) to list and describe problem types that various operations of a service may emit in error responses. In short, a consumer can not handle error scenarios and write a robust client-side code unless the API developer has put an extra effort in listing error types with the API contract or documentation using some other means.

### Externalize

Often problem details emitted by APIs are written by developers and make a lot of assumptions about the context and the audience. It is hard to change such details if these are hardcoded in implementation of services exposing APIs. For example, to change a message from

`Add Card refused due to compliance guidelines` to

`Could not add card due to failure to comply with guideline {}`

a developer has to make change(s) in the code and redeploy the service(s), a costly affair. This could be avoided if the messages are externalized from the code.

### Localize error message strings

If error related strings such as `title`, `detail`, `actions`, etc. are externalized, it is easy for the documentation and an internationalization teams to localize these without any help from the service developers and without requiring redeployment of the services.

## Error Catalog Service API

Error Catalog Service API provides operations to manage error types and error catalogs that are associated with HTTP APIs. Error Type relates to the Problem Type of RFC 7807. 


### API

The API for Error Catalog Service is defined using the OpenAPI Specification. The API contract can be found in [error-catalog-service.json](error-catalog-service.json).

### Resources

There are two resources defined in the Error Catalog Service.

1. Error Type
2. Error Catalog

#### Error Type

Error Type describes an error. Error Type relates to the Problem Type of RFC 7807. An Error Type can be associated with one or more operations that belong to one or more APIs. Error Type [schema](./models/error_type.json) includes an identifier for the error type, details about the error, HTTP status code(s) that could be used to inform about this error and it describes possible action(s) to take in order to resolve the error.

`/v1/error/error-catalogs/{ecid}/error-types`


#### Error Catalog

An Error Catalog is a collection of error types. In addition, each catalog is associated with an API namespace and language. There SHOULD be an Error Catalog for a default language (e.g. `en_US`). There could be one or more language-specific error catalogs for the same API namespace. Error Catalog schema is defined in [error_catalog.json](./models/error_catalog.json)

`/v1/error/error-catalogs`

### Use Cases

We anticipate the following use cases for the Error Catalog Service.

1. Services exposing APIs can retrieve relevant error types from the Error Catalog Service instead of hard-coding error strings in the code.
2. Technical writers can use a tool to curate error catalogs using the Error Catalog Service.
3. API portals can show a list of error types associated with an API using the Error Catalog Service and suggested OpenAPI [Extension for Error Type](./OpenAPIExtensionForErrorType.md).
4. Linters can validate if an API contract annotated with the suggested extension relates to correct error types in the Error Catalog Service.
4. API automation tests can validate if the error responses emitted by the service implmenting the API are as documented in the API contract annotated with the suggested extension.

### Naming Conventions

We have used *snake_case* for naming properties in JSON schema for resources and error responses. Feel free to use your own conventions as required.

### Namespace

Error Catalog Service uses `error` as an API namespace as can be seen from `path` below.

`/v1/error/error-catalogs`


### Error response

The [error.json](./models/error.json) and [error_instance.json](./models/error_instance.json) are the schemas used for error responses in Error Catalog Service API. 

These schemas are derived from [Problem Details JSON Object](https://tools.ietf.org/html/rfc7807#section-3) as described in RFC 7807. We have added a few extensions such as `id`, `instances` and `links`. We have used an array of `instances` instead of an `instance` since `400` `Bad Request` types of errors may require to report more than one instances in error responses. Schema for error_instance.json is also derived from standard validation output format as defined in [section 10](https://json-schema.org/draft/2019-09/json-schema-core.html#rfc.section.10) of JSON Schema specification. 

In case, you have a more appropriate way to carry an error in a format that your API already defines, feel free to use that format instead of [error.json](./models/error.json).


### Annotated with OpenAPI Extension

The [API contract](error-catalog-service.json) for the Error Catalog Service itself is annotated with the Error Type extension as defined in [OpenAPIExtensionForErrorType](./OpenAPIExtensionForErrorType.md).

**Note**: Since API described has no service implementation, the API contract uses a fictitous variable called `{ecid}` as error catalog id.

## Community

* Contribute to the API design via issues and pull requests.
* Contribute implementation of the Error Catalog Service in your favorite programming language.
* Contributors: people and organizations who helped us get started or are actively working on the definition of the Error Catalog Service API, various extensions and implementation of the service in various languages.
* Demos & open source -- if you have something to share about your use of Error Catalog Service, please submit a PR!

## References

1. [Error Catalog - PayPal Design Guidelines](https://github.com/paypal/api-standards/blob/master/api-style-guide.md#error-catalog)
2. [JSON Schema: A Media Type for Describing JSON Documents](https://tools.ietf.org/html/draft-handrews-json-schema-02)
3. [OpenAPI Specification](https://swagger.io/specification/)
4. [Problem Details for HTTP APIs, RFC 7807](https://tools.ietf.org/html/rfc7807)
5. [OpenAPI Extension for Error Type](./OpenAPIExtensionForErrorType.md)
6. [Microsoft Azure Cluster Errors](https://docs.microsoft.com/en-us/azure/hdinsight/create-cluster-error-dictionary)
7. [Twilio Error and Warning Dictionary](https://www.twilio.com/docs/api/errors)
8. [API Style Book, Error Format](http://apistylebook.com/design/topics/http-status-standard-error)
