# Canary Policy for Mule 4
A custom policy to perform canary releases, intercepting the incoming calls and deciding which implementation URL to route the call to. Applying this policy to your API or proxy you would be able to:
  - Define an array of endpoints, with their weights
  - Enable session stickiness (TO-DO), to keep record of previous redirections based on a customizable header
  - Capture metrics for the incoming events
  - Get the stored metrics

### Why?
A canary release helps organizations to reduce the risks of introducing new versions of a software by incrementally rolling out traffic to the new version, improving the [observability](https://en.wikipedia.org/wiki/Observability) and limiting the impact of the new components over the existing service.

### How?
For a canary release to exist, the following elements should be present:
- An old version of the software, a REST API in this context
- A new version of the API, called the canary version (can be multiple canaries)
- API Consumers. They can be either the same types of consumers or different ones
- A routing criteria. This can be based on the type of audience, I.e. “consumers of type A will go to the old version, while consumers of type B will be canary consumers or early adopters, being redirected to the Canary version” or based on traffic, I.e. “90% of the traffic will be sent to the previous version, while the remaining 10% will be redirected to the canary version”
- A router, that based on the routing criteria, redirects the traffic to the available endpoints  

### Deployment Architecture

There are no limitations imposed by the use of this policy regarding the deployment architecture, except the need of having two artifacts (applications) deployed in Anypoint Runtime Manager (one for each version). However, the following topology is recommended as it provides a more flexible solution in terms of deprecation and retirement and, also, improves the [observability](https://en.wikipedia.org/wiki/Observability)

![](./images/deployment.png "Deployment Architecture")

From the above, a proxy is deployed on top of both versions (original and canary) in order to centralize communication, providing an abstraction and improving understanding from the point of view of networking and traffic management. Please see ["Limitations"](###Limitations) and ["Known Issues"](###Known-Issues) sections.

If you want to skip the extra layer added by the proxy, you can always apply the policy on top of the original (baseline) application:

![](./images/deployment-nr.png "Deployment Architecture - Not Recommended")

This option requires to set an additional flag "appliedOnApi". Please see ["Usage"](###Usage).

But this approach may lead to a management nightmare, where deprecation and retirement of APIs become an almost impossible task. See ["Deprecation and Retirement"](###Deprecation-and-retirement) section.

### Deprecation and retirement
Ask yourself: What do I want to do to discontinue the original version of my API when the Canary version has been tested and is ready to be used as current version?
Here are a series of strategies for that end:
- "Increase the weight of my Canary version to 100%, so that the traffic is only redirected there". This is an option that the only thing it achieves is an ease in the configuration, but under no point of view it is the optimal solution, since adopted in a proxy, it will generate an inconsistency between the implementation url configured in the proxy and the real url. Even worse if this strategy is used when applying the policy on the original API (no proxy), since this component will not only consume unnecessary resources, but it cannot be removed to ensure the existence of the policy that makes the routing.
- "Remove the policy from the Proxy and change the implementation url from original to canary". This is a valid option, but keep in mind that it depends on how the proxy is accessed, because it can mean a breaking change for the consumer. This option is very valid for when the canaries strategy is limited to deploying minor changes that do not impact the versioning strategy of the API spec (build numbers, for instance v1.1, v1.2, both under same path /v1).
- "Remove the proxy, along with the policy": It is the cleanest way to proceed, but the most complex and risky. Requires that the canary application be redeployed first as the original app (if it is a non breaking change) or leave it as is (for major changes, breaking) and make the normal changes expected as part of the normal API SDLC

### Usage
After publishing to Exchange, follow these steps to apply the policy to an existing managed API (or proxy):

* Log into Anypoint Platform
* Enter API Manager
* Click on the API version for the application you want to apply the policy to
* Click on Policies
* Click on Apply New Policy
* Filter by 'Custom' category and select 'Canary Release (Mule 4)'. Click on 'Configure Policy' button
* Give value to the policy's parameters:

| Parameter | Purpose |
| ------ | ------ |
| Host (Original) | Details the host for the original version. This should be the same as the one set on the implementation url if using a proxy |
| Port (Original) | Details the port for the original version |
| Protocol (Original) | Details the protocol for the original version |
| Path (Original) | Details the path for the original version |
| Weight (Original) | Details the weight for the original version. Represents a percentage that is calculated taking into account a sample of 10 requests. For example: 50 indicates that 5 requests out of 10 will be routed to this endpoint |
| Host (Canary) | Details the host for the canary version |
| Port (Canary) | Details the port for the canary version |
| Protocol (Canary) | Details the protocol for the canary version |
| Path (Canary) | Details the path for the canary version |
| Weight (Canary) | Details the weight for the canary version. Represents a percentage that is calculated taking into account a sample of 10 requests. For example: 50 indicates that 5 requests out of 10 will be routed to this endpoint |
| appliedOnApi | Select this option only if the policy is applied on the base API (original) instead of on a proxy. This forces the <http-policy:execute-next> directive. See https://docs.mulesoft.com/api-manager/2.x/custom-policy-4-reference#basic-xml-structure for further details. |
| sessionStickinessSupport | Indicates if the session stickiness capability is enabled (TO-DO) |
| Session Stickiness Header Expression | If session stickiness is enabled, this DW expression represents how to access the header that contains the key used to track the session stickiness (TO-DO) |
| Override Object Store Settings? | Select this option to override the default Object Store. The default is false. |
| Is the Object Store persistent? | If checked, uses a persistent Object Store ) |
| Object Store's entry TTL | The entry timeout. Default value is 1 (hour) ) |
| Object Store's entry TTL unit | The time unit. Default value is "HOURS". ) |


#### Development

The following commands are required during development phase

| Task | Command |
| ------ | ------ |
| Package policy | mvn clean install |
| Publish to Exchange - Make sure to update the pom.xml file with your org ID - | mvn deploy |

### Debugging
The following package can be added to the log4j2.xml configuration ```com.mule.policies.canary```
In Debug mode, it will print the following checkpoints:
- Weights array print. This is important to understand how the array was created based on the input parameters
- Current status of the Traffic Object Store, used to keep track of the metadata of the requests (only if is the first time you're creating it)
- Incoming Request attributes, before routing the endpoints
- Destination Endpoint. Wether it is the original or the canary URL
- Flag indicating the Traffic Object Store will be updated

### Metrics
The policy incorporates the possibility of collecting usage metrics to later be used in a Canary Analysis process.

#### To enable the Metrics
- Enable "Capture Raw Usage Metrics?" by clicking on the radio button, while applying the policy
- Complete the extra fields enabled when you clicked on the above:

| Parameter (Internal Name)| UI Name | Purpose |
| ------ | ------ | ------ |
| metricsOsIndex | Index used to store the metrics | Index used to store the metrics. By default uses a unique ID per event |
| metricsOsPersistent | Is the Metrics Object Store persistent? | Flag to configure persistent OS |
| metricsOsTtl | Metrics Object Store entry TTL | Time to live for the OS (this is applicable either for In-memory and Persistent OS) |
| metricsOsTtlUnit | Metrics Object Store entry TTL unit | Time unit for the above TTL |

#### To capture the stored Metrics
Simply send a GET to the API with `/metrics` endpoint

#### Metrics Structure
The provided metrics follow a java formatted event:
```
"canary": "0",
"correlationId": "3eb2ee40-5c22-11ec-bf1d-0284a1451db2",
"responseTimeMs": 1191,
"statusCode": 200
```
where:
- canary: indicates if the event is associated with the canary URL
- correlationId: indicates the correlationId assigned by Mule to the balanced request (not the one associated to the `/metrics` endpoint)
- responseTimeMS: indicates the total response time (in millis) for the event. This is calculated as `(end time - start time)`. The start time is calculated as the moment when the policy receives the traffic. The end time is calculated as the moment when the policy receives a response from the underlying API.
- statusCode: indicates the HTTP Status Code for the event

#### *IMPORTANT*
- Take into account that the underlying API shouldn't contain an endpoint called `/metrics`, otherwise that logic won't be executed and might interfere with existing processes. If your API already exposes an `/metrics`, please make sure to change the logic of this policy to rename the endpoint used to extract the metric
- Take into account the current usage of your API, the metrics captured, the reporting tools in place, the existing policies applied (i.e. Rate Limiting) and the app config before enabling the metrics, as if implemented as they should, the policy will receive a lot of requests and generate a lot of events
- The current implementation is not taking into account any strategy to clean the old events


### Limitations
NOTE: The recommended approach to manage Canary Releases should be having a third party component, outside Anypoint Platform, specially designed to handle this kind of needs. For instance, nginx provides a module called [split clients](https://nginx.org/en/docs/http/ngx_http_split_clients_module.html?_ga=2.76046677.1103157284.1620664242-1521291711.1620664242 ), useful to assign percentages of traffic that we want to redirect to defined clients (hosts). The solution provided here is a custom solution, provided by Professional Services and and that may not have official product support.

If you want to use this solution anyway, this approach leads to the following problems (may or may not be applicable to your organization):
- The networking team is the one who normally manage traffic to services. This policy would change that if applied by Developers, making it complex to define a clear ownership model, understand and manage it. Furthermore, these teams will need access to applications in order to handle the changes
- The routing would be done from a component that is applied on top of a mule application or proxy. Not a device specially designed for this purpose such as, for example, a Load Balancer
- In case of the traffic weight algorithm, the solution needs to store metadata of each request, and that metadata should be available for all the members of a cluster (servers, workers, pods). The storage is managed by the Object Store module, so please consider the technical challenges that the use of this module implies
- Extra component is required (proxy), adding more hops to the process -OPTIONAL if proxy approach was chosen-. In a typical API-Led approach, having an additional step on top of the existing 3 layers might be counterproductive with defined SLAs and SLOs
- Adds environment promotion complexity as part of the CI/CD pipelines
- Deprecation and Retirement becomes a time consuming task (more than usual!)
- The underlying proxy logic is never used. ```<http-policy:execute-next />``` is never invoked.

### Contribution

Want to contribute? Great!

* For public contributions - Just fork the repo, make your updates and open a pull request!
* For internal contributions - Use a simplified feature workflow following these steps:
   - Clone the repo
   - Create a feature branch using the naming convention feature/name-of-the-feature
   - Once it's ready, push your changes
   - Open a pull request for a review
