# SpringBoot

### References

* [Swagger: API first approach](https://swagger.io/resources/articles/adopting-an-api-first-approach/)

### Introduction
* Issues with monolithic architecture were specific to the product.
* When we move from monolithic architecture to microservices, we get a new set of problems that are more generic (load balaning, service discovery etc.). These new set of problems, have well documented frameworks/paradigms to tackle them.

### Beans
* Beans are used when you need a long lived instance (mostly the lifetime of the service) of an object.
* Beans are singleton. Beans have a single instance of an object running, and keep getting reused.
* Dependent classes use *Dependency Injection* to access the instance of the bean. (eg using `@Autowired`).
* Beans can be created using many annotations. eg `@Bean`, `@Component`, `@Service`, `@Controller` etc.

```java
  class Foo {
    @Bean
    public RestTemplate getRestTemplate() {
      	return new RestTemplate();
    }
  }
  
  class Bar {
    	@Autowired
    	private RestTemplate restTemplate;
    
    	public void test() {
        	// Doesn't have to initialize or construct a RestTemplate object.
        	restTemplate.exchange(...);
      }
  }
```

### Swagger-SDK
* API as first class objects:
* codegen can be used to create yaml to model, and the `swagger-sdk` model classes are used as dependency in server project.
* the codegen is generated from `server/**/resources/public/*.yaml`, ie the public APIs defined for the service. (eg `DoSomethingRequest`: yaml object that defines the request body for `doSomething`)
* swagger-sdk can also has `src/java/**/*.java` files to add more supporting classes, generally used for exposing internal classes that are not needed in API documentation, but maybe needed for using the SDK (for example, model files).
* Both yaml (public class) and internal java files (internal class) are avaible in SDK. The SDK is used by `./server` and `./server`.
* `api-doc-external.yaml` is the external API endpoint. It is used to create the documentation for REST APIs and also generates "client" classes and function to use these API through an SDK.

```yaml
  /orgs/{orgId}/deployments/{deploymentId}/object/{objectId}/operations/doSomething:
post:
  tags:
    - Foo Bar Service
  summary: Do Something
  description: Do this something for specified object
  operationId: doSomething
  parameters:
    - $ref: '#/parameters/orgIdPathParam'
    - $ref: '#/parameters/sddcIdPathParam'
    - $ref: '#/parameters/objectIdPathParam'
  responses:
    '200':
      description: OK
      schema:
        $ref: '#/definitions/Operation'
```
gets converted into something like:

```java
public class FooBarServiceApi 
		public Operation doSomething(UUID orgId, UUID sddcId, UUID objectId) throws RestClientException {
    		uriVariables.put("orgId", orgId);
    		uriVariables.put("sddcId", sddcId);
    		uriVariables.put("objectId", objectId);
    		String path = UriComponentsBuilder.fromPath(
          		"/orgs/{orgId}/deployments/{deploymentId}/object/{objectId}/operations/doSomething")
          	.buildAndExpand(uriVariables)
          	.toUriString();
		}
}
```
* So, finally the SDK has 3 main components:
  * `com.vmware.hercules.foobar.client.api.FooBarServiceApi`: These are the client APIs in the SDK that can be used to call the REST endpoint programatically.
  * `com.vmware.hercules.foobar.model`
    * Internal classes that were present in `swagger-sdk/**/src`
    * External classes generated from API defs yaml files in `server/**/resources/public`

### Marshalling/Unmarshalling Java Objects 
* The client SDKs, using `RestTemplate`, perform REST operations. These REST operations return json strings.
* These strings are actually objects in string form, that need to be deserialized into java objects.
* RestTemplate has a mechanism of doing this. `RestTemplate::getForObject(url, class)` can be used to perform a get operation and serialize the output to a class.
* The server and the client need to share this class. They do this by either:
  * Copying the implementation of this class from server to the client (duplicate but independent code).
  * server publishes the SDK using `swagger-sdk` with these classes present in `model`