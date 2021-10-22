# Circuit-Breaker Custom Policy

This is a custom policy that implements a lightweight Circuit Breaker pattern for Mule 4. By applying this policy to an API, these features are added:

  - Define an error threshold giving the API the flexibility to fail as many times as you define before tripping the circuit
  - Define a retry period after which the protected API should allow incoming requests
  - Set the circuit to HALF-OPEN state after the threshold is reached and service is still not functional
  - Perform dynamic exception handling

### Why?
When working on a layered architecture (API-Led is a good example) it doesn't make sense to propagate the incoming requests when we know that one of the components of this architecture is not working correctly. This policy provides an entry point for the consumer, preventing spreading calls through the different layers, allowing time to failing resources to recover.

### How?
This policy handles a deterministic model that indicates the state of the circuit. It uses the Mule Object Store (OS) to save and retrieve the values after each call.

When the application starts, OS is initialized using the ${appId} property as key, as shown below.

![](./docs/images/cbstore.png)

This ensures that every application that uses this policy has isolated circuit state values.

*NOTE:* OS settings can be overridden if needed when configuring the policy.

## Circuit Breaker Design

**State Transition**

![](./docs/images/states-transition.png)

- The policy starts with the circuit in `CLOSED` state, which is the non-error state that allows calls to the API.
- If the underlying service throws an error, the policy counts it until the maximum number of errors allowed set by the Failure Threshold is reached, then the policy trips the circuit, which is the `OPEN` state.
- When the policy is in an `OPEN` state, it immediately rejects a new incoming request, unless the timestamp of the last error incremented by the Retry Period exceeds the current timestamp. In that case, the policy transitions to `HALF-OPEN` state and propagates the incoming request.
- When the policy is in a `HALF-OPEN` state, if the policy receives an error from the last request, it increments the counter and transitions back to `OPEN` state; otherwise, the policy clears the error counter and transitions to `CLOSED` state.

**Sequence Diagram**

![](./docs/images/sequence.png)

## Setup

After publishing to Exchange, follow these steps to apply the policy to an existing managed API (or proxy):

* Log into Anypoint Platform
* Enter API Manager
* Click on the API version for the application you want to apply the policy to
* Click on Policies
* Click on Apply New Policy
* Filter by 'Custom' category and select 'circuit-breaker-mule-4'. Click on 'Configure Policy' button
* Give value to the policy's parameters:

| Parameter | Purpose |
| ------ | ------ |
| Failure Threshold | maximum number of errors allowed before opening the circuit (putting it in OPEN state) |
| Retry Period | amount of time to wait before allowing requests into the API after opening a circuit |
| Retry Period Unit | the time unit for the retry period value: seconds, minutes, or hours |
| Evaluate the error object to trigger the circuit? | - Checkbox - Select this option if the underlying application propagates the error object. E.g. applications not handling errors or raising custom ones on error handling strategies |
| Exceptions Array | a comma separated string containing the exception types that are expected to trip the circuit. Example: "MULE:COMPOSITE_ROUTING, HTTP:UNAUTHORIZED, MULE:EXPRESSION" |
| Evaluate the HTTP response to trigger the circuit? | - Checkbox - Select this option to evaluate the HTTP response (status code). Check this option if evaluate error object option is unchecked.    |
| HTTP Codes Array | Specify all the HTTP codes that can trip the circuit. The default value is 500 (Internal Server Error). Expect a comma separated string. Example: "500, 401". Double quotes are required but spaces between types are not. |
| Include the circuit breaker info in the response body? | - Checkbox - Select this option to get circuit information in the response body, which is for troubleshooting. |
| Override Object Store settings? | - Checkbox - Select this option to override default OS settings. Defaut OS will use persistent OS, with 1 hour entry TTL. |
| Object Store's entry TTL | The entry timeout. Default value is 1 (hour). |
| Object Store's entry TTL unit | The time unit. Default value is "HOURS". You can choose one of the listed options based on https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/TimeUnit.html|

This policy can be applied at the resource level or across the entire API.

## Usage

The policy only engages the circuit breaker functionality if `Evaluate the error object to trigger the circuit?` or `Evaluate the HTTP response to trigger the circuit?` is checked, and the corresponding criteria is met.  If neither of those are checked, or the criteria is not met, then the API executes normally without any modification from the policy.

When the circuit is open, the policy returns the `retry-after` header, which is defined in [RFC 2616][retryAfterSpec].  This defines the time, in [HTTP-date format][httpDateSpec], when the circuit breaker will be half-open and allow a call into the API.  This header is not returned if the circuit is closed or half-open.  An example is provided below.

```
retry-after:Fri, 22 Oct 2021 11:28:21 -05:00
```

The caller can determine if the circuit breaker is preventing API access by the HTTP response containing the `503` status code and the `retry-after` header.  When the `retry-after` header is in the response, the caller should wait to make another call until after the time in the header.

### Responses
The policy includes several items in the response when triggered.

- **HTTP Status Code**: When the circuit breaker is triggered (criteria is met or the circuit is open), the policy always returns HTTP status code `503`.
- **HTTP Header**: When the circuit is open, the policy returns the `retry-after` header.
- **HTTP Body**: The policy returns different things in the body, depending on the configuration.
  - *Open Circuit*: When the circuit is open, the policy returns an empty body.
  - *Exception Check*: When the policy is handling API exceptions and receives an exception in the list, it adds the `error` field to the response body that contains the exception's description. 
  - *Status Code Check*: When the policy is checking API status codes and receives a code in the list, it returns the API's response body as the response body.  If the API's response body is empty (null, empty string, empty array, empty object), then the policy returns an empty response body.
  - *Circuit Breaker Info*: If the circuit breaker information checkbox is checked, and the circuit breaker is triggered, then the `circuitBreaker` field is added to the response body.  If there is already a response body, it is merged with that; because the body can be any format which may be incompatible with the circuit breaker, the existing body is moved to the `data` field.

**Empty Body**
An empty response body.

```
HTTP/1.0 503 Service Unavailable
Content-Type:application/json; charset=UTF-8
retry-after:Fri, 12 Nov 2019 09:60:02 -05:00
transfer-encoding:chunked
Connection:keep-alive
```

**Empty Body + Circuit Breaker Info**
Circuit breaker info is added to an empty response body.

```
HTTP/1.0 503 Service Unavailable
Content-Type:application/json; charset=UTF-8
retry-after:Fri, 12 Nov 2019 09:60:02 -05:00
transfer-encoding:chunked
Connection:keep-alive
{
    "circuitBreaker": {
        "failureThreshold": 3,
        "retryPeriod": 20,
        "retryPeriodUnit": "S",
        "state": "OPEN",
        "timestamp": "2019-11-12T14:59:42.942Z",
        "errorCount": 4,
        "retryAfter": "2019-11-12T14:60:02.942Z"
    }
}
```

**Exception Check**
Body is description of exception from API.

```
HTTP/1.0 503 Service Unavailable
Content-Type:application/json; charset=UTF-8
transfer-encoding:chunked
Connection:keep-alive
{
    "error": "This is the exception description."
}
```

**Exception Check + Circuit Breaker Info**
Circuit breaker info is added to a response body that contains the error description.

```
HTTP/1.0 503 Service Unavailable
Content-Type:application/json; charset=UTF-8
transfer-encoding:chunked
Connection:keep-alive
{
    "error": "This is the exception description.",
    "circuitBreaker": {
        "failureThreshold": 3,
        "retryPeriod": 20,
        "retryPeriodUnit": "S",
        "state": "CLOSED",
        "timestamp": "2019-11-12T14:59:42.942Z",
        "errorCount": 1
    }
}
```

**Status Code Check**
Body is existing response from API.

```
HTTP/1.0 503 Service Unavailable
Content-Type:application/json; charset=UTF-8
transfer-encoding:chunked
Connection:keep-alive
{
    "message": "This is the response body returned by the API."
}
```

**Status Code Check + Circuit Breaker Info**
Circuit breaker info is added to a response that already has a body.  API body is moved to `data` field for compatibility.  Status code received from API is added to circuit breaker info.

```
HTTP/1.0 503 Service Unavailable
Content-Type:application/json; charset=UTF-8
transfer-encoding:chunked
Connection:keep-alive
{
    "data": {
        "message": "This is the response body returned by the API."
    },
    "circuitBreaker": {
        "failureThreshold": 3,
        "retryPeriod": 20,
        "retryPeriodUnit": "S",
        "state": "CLOSED",
        "timestamp": "2019-11-12T14:59:42.942Z",
        "errorCount": 1,
        "errorStatusCode": "500"
    }
}
```

#### Circuit Breaker Info

When circuit breaker info is provided in the response body, it contains the fields below.  Some only show in the conditions described.

- `failureThreshold`: The maximum number of errors allowed before tripping the circuit.
- `retryPeriod`: The amount of time to wait before allowing requests into the API after opening a circuit.
- `retryPeriodUnit`: The time unit code for the retry period.
- `state`: Current state of the circuit.
- `timestamp`: The date/time of the request.
- `errorCount`: The number of requests that has been sent to the API and failed.
- `errorStatusCode`: When checking status codes, this is the status code received from the API which triggered the circuit breaker.  It is in the policy's status code list.
- `retryAfter`: When the circuit is open, this is date/time when a request will be allowed again.

### Troubleshooting

If the policy is not behaving as expected, developers can follow the troubleshooting options below.

- Enable circuit breaker info in the response body by checking `Include the circuit breaker info in the response body?` checkbox.  This allows the caller to see detailed circuit breaker state in the response body.
- Enable the policy's loggers.  This allows the operator to see detailed circuit breaker state in the log.  This can be done by adding the following line to log4j2.xml or equivalent depending on infrastructure: `<AsyncLogger name="com.mule.policies.circuitbreaker" level="DEBUG"/>`.

#### Development

The following commands are required during development phase

| Task | Command |
| ------ | ------ |
| Package policy| mvn clean install |
| Publish to Exchange | mvn deploy |

##### Dependencies
This policy uses a persistent Object Store as a key value database that allows maintaining the state of the circuit at every time. It also uses the http transport extension module, to perform the update of headers in the response in case of error.

### Contribution

Want to contribute? Great!

* For public contributions - Just fork the repo, make your updates and open a pull request!
* For internal contributions - Use a simplified feature workflow following these steps:
   - Clone your repo
   - Create a feature branch using the naming convention feature/name-of-the-feature
   - Once it's ready, push your changes
   - Open a pull request for a review

### Todos
 - Write Tests


[retryAfterSpec]: https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.37
[httpDateSpec]: https://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.3.1