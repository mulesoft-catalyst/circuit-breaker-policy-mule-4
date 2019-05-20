# Circuit-Breaker Custom Policy

This is a custom policy that implements a lightweight Circuit Breaker pattern for Mule 4. Applying this to your API you would be able to:

  - Define an error threshold giving your API the flexibility to fail as many times you defined before tripping the circuit
  - Define a retry period after which the protected API should allow incoming requests.
  - Define a timeout for the incoming requests (WIP)

### Why?
When working on layered architecture (API Led is a good example) it doesn't make sense to propagate the incoming requests when we know that some component of this architecture is not working correctly. This policy provides an entry point for the consumer, preventing spreading calls through the different layers, giving time to failing resources to recover.

### How?
This policy handles a deterministic model that indicate the state of the circuit. It uses the Mule Object Store to save and retrieve the values after each call.


### Usage
After publishing to Exchange, follow these steps to apply the policy to an existing managed API:

* Log into Anypoint Platform
* Enter API Manager
* Click on the API version for the application you want to apply the policy to
* Click on Policies (at your left. No! your other left :P)
* Click on Apply New Policy
* Filter by 'Custom' category and select 'circuit-breaker-mule-4'. Click on 'Configure Policy' button
* Give value to the policy's parameters:

| Parameter | Purpose |
| ------ | ------ |
| timeout| WIP.see todo's. Specify the maximum number of seconds that a consumer can wait after sending a request|
| failureThreshold | maximum number of errors allowed before tripping the circuit (putting it in OPEN state) |
| retryPeriod | number of seconds the pattern will wait before trying to reach depedent components (underlying APIs) when a new request is received |

#### Development

The following commands are required during development phase

| Task | Command |
| ------ | ------ |
| Package policy| mvn clean install |
| Publish to Exchange | mvn deploy |

##### Dependencies
Esta policy utiliza Object Store como base de datos clave valor que permite mantener el estado del circuito en todo momento. Ademas utiliza http transport module extension, para realizar la inyeccion de headers en la respuesta en caso de error.

### Contribution

Want to contribute? Great!

Just fork the repo, make your updates and open a pull request!

### Todos
 - Add timeout capability
 - Write Tests
 - Add intemediate HALF-OPEN state for the circuit
 - Improve performance
 - Add capabilities, like dynamic exception handling (currently the policy is catching only ANY error types)

License
----
MIT