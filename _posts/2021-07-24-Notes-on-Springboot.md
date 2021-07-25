### References
* [Swagger: API first approach](https://swagger.io/resources/articles/adopting-an-api-first-approach/)
* [Spring Framework definition](https://docs.spring.io/spring-framework/docs/current/reference/html/)
* [DispatcherServelet](https://docs.spring.io/spring-framework/docs/3.0.0.M4/spring-framework-reference/html/ch15s02.html)
* [Spring vs SpringBoot] (https://www.baeldung.com/spring-vs-spring-boot)
### Keywords
* **Spring Framework**
  * A framework that lets you write Java Enterprise applications.
  * At the heart are the modules of the core container, including a configuration model and a dependency injection mechanism.
  * Provides foundational support for different application architectures, including messaging, transactional data and persistence, and web.
  * It also includes the Servlet-based `Spring MVC` web framework and, in parallel, the `Spring WebFlux` reactive web framework.
  * **Spring Framework’s Inversion of Control (IoC) container**
    * IoC is also known as dependency injection (DI).
    * It is a process whereby objects define their dependencies (that is, the other objects they work with) only through constructor arguments, arguments to a factory method, or properties that are set on the object instance after it is constructed or returned from a factory method.
    * The container then injects those dependencies when it creates the bean.
    * The following example shows the basic structure of XML-based configuration metadata. This configuration metadata represents how you, as an application developer, tell the Spring container to instantiate, configure, and assemble the objects in your application.
    {% highlight xml linenos %}
    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://www.springframework.org/schema/beans
            https://www.springframework.org/schema/beans/spring-beans.xsd">
        <bean id="..." class="...">
            <!-- dependencies and configuration for this bean go here -->
        </bean>
        <bean id="..." class="...">
            <!-- dependencies and configuration for this bean go here -->
        </bean>
        <!-- more bean definitions go here -->
    </beans>
    {% endhighlight %}
  * **Beans**
    * A Spring IoC container manages one or more beans.These beans are created with the configuration metadata that you supply to the container (for example, in the form of XML `<bean/>` definitions).
    * Within the container itself, these bean definitions are represented as `BeanDefinition` objects.
    * A bean definition is essentially a recipe for creating one or more objects.
    * The container looks at the recipe for a named bean when asked and uses the configuration metadata encapsulated by that bean definition to create (or acquire) an actual object.
  * Spring framework gives access to a lot of modules that make writing production level code easier.
  * For example, in the early days of Java web development, we needed to write a lot of boilerplate code to insert a record into a data source. By using the `JDBCTemplate` of the `Spring JDBC` module, we can reduce it to a few lines of code with only a few configurations.
  * Spring requires defining the dispatcher servlet, mappings, and other supporting configurations. We can do this using either the `Deployment Descriptor` file `web.xml` or an `Initializer` class.
  * **Example Spring Application using WebApplicationInitializer**
    {% highlight java linenos %}
    public class MyWebAppInitializer implements WebApplicationInitializer {
      @Override
      public void onStartup(ServletContext container) {
          AnnotationConfigWebApplicationContext context
            = new AnnotationConfigWebApplicationContext();
          context.setConfigLocation("com.baeldung");
          container.addListener(new ContextLoaderListener(context));
          ServletRegistration.Dynamic dispatcher = container
            .addServlet("dispatcher", new DispatcherServlet(context));
          dispatcher.setLoadOnStartup(1);
          dispatcher.addMapping("/");
      }
    }
    {% endhighlight %}
    * We also need to add the `@EnableWebMvc` annotation to a `@Configuration` class, and define a view-resolver to resolve the views returned from the controllers:
    {% highlight java linenos %}
    @EnableWebMvc
    @Configuration
    public class ClientWebConfig implements WebMvcConfigurer {
       @Bean
       public ViewResolver viewResolver() {
          InternalResourceViewResolver bean
            = new InternalResourceViewResolver();
          bean.setViewClass(JstlView.class);
          bean.setPrefix("/WEB-INF/view/");
          bean.setSuffix(".jsp");
          return bean;
       }
    }
    {% endhighlight %}
  * **Example Spring Application using web.xml**
    * Controller: `HelloController.java`
    {% highlight java linenos %}
    package com.tutorialspoint;
    import org.springframework.stereotype.Controller;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RequestMethod;
    import org.springframework.ui.ModelMap;
    @Controller
    @RequestMapping("/hello")
    public class HelloController {
       @RequestMapping(method = RequestMethod.GET)public String printHello(ModelMap model) {
          model.addAttribute("message", "Hello Spring MVC Framework!");
          return "hello";
       }
    }
    {% endhighlight %}
    * `web.xml`: Notice we are mapping `HelloWorld` servelet with `/` url pattern. A web container can have multiple such servelets for for different url patterns, all defined statically in the deployment descriptor `web.xml` file
    {% highlight xml linenos %}
    <web-app id = "WebApp_ID" version = "2.4"
       xmlns = "http://java.sun.com/xml/ns/j2ee"
       xmlns:xsi = "http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation = "http://java.sun.com/xml/ns/j2ee
       http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">
       <display-name>Spring MVC Application</display-name>
       <servlet>
          <servlet-name>HelloWeb</servlet-name>
          <servlet-class>
             org.springframework.web.servlet.DispatcherServlet
          </servlet-class>
          <load-on-startup>1</load-on-startup>
       </servlet>
       <servlet-mapping>
          <servlet-name>HelloWeb</servlet-name>
          <url-pattern>/</url-pattern>
       </servlet-mapping>
    </web-app>
    {% endhighlight %}
    * Finally, the Spring configuration of the servelet in the `HelloWeb-servlet.xml`:
    {% highlight xml linenos %}
    <beans xmlns = "http://www.springframework.org/schema/beans"
       xmlns:context = "http://www.springframework.org/schema/context"
       xmlns:xsi = "http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation = "http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context-3.0.xsd">
       <context:component-scan base-package = "com.tutorialspoint" />
       <bean class = "org.springframework.web.servlet.view.InternalResourceViewResolver">
          <property name = "prefix" value = "/WEB-INF/jsp/" />
          <property name = "suffix" value = ".jsp" />
       </bean>
    </beans>
    {% endhighlight %}
    * and the following is the Spring *view* `.jsp`  file
    {% highlight html linenos %}
    <%@ page contentType = "text/html; charset = UTF-8" %>
    <html>
       <head>
          <title>Hello World</title>
       </head>
       <body>
          <h2>${message}</h2>
       </body>
    </html>
    {% endhighlight %}
    * Spring bootstrapping
      * `Servlet` container (the server) reads `web.xml`.
      * The `DispatcherServlet` defined in the `web.xml` is instantiated by the container.
      * `DispatcherServlet` creates `WebApplicationContext` by reading `WEB-INF/{servletName}-servlet.xml`.
      * Finally, the `DispatcherServlet` registers the beans defined in the application context.
      * `com.tutorialspoint` package is scanned, a controller class is found that can respond to web requests.
* **Spring Boot**
  * Makes it easy to create (bootstrap) standalone production grade Spring application.
  * Chooses convention over configuration to bootstrap an application that just runs.
  * Spring Boot offers a fast way to build applications. It looks at your classpath and at the beans you have configured, makes reasonable assumptions about what you are missing, and adds those items. With Spring Boot, you can focus more on business features and less on infrastructure.
    * Is Spring MVC on the classpath? There are several specific beans you almost always need, and Spring Boot adds them automatically. A Spring MVC application also needs a servlet container, so Spring Boot automatically configures embedded Tomcat.
    * Is Jetty on the classpath? If so, you probably do NOT want Tomcat but instead want embedded Jetty. Spring Boot handles that for you.
    * Is Thymeleaf on the classpath? If so, there are a few beans that must always be added to your application context. Spring Boot adds them for you.
  * Spring applications generally create a WAR file that then needs to be deployed to a servlet (eg tomcat) container. Springboot instead creates a standalone application with embedded webserver in it (no need for a separate servlet container).
  * Spring Boot does not generate code or make edits to your files. Instead, when you start your application, Spring Boot dynamically wires up beans and settings and applies them to your application context.
  * The entry point of a Spring Boot application is the class which is annotated with @SpringBootApplication:
  {% highlight java linenos %}
  @SpringBootApplication
  public class Application {
      public static void main(String[] args) {
          SpringApplication.run(Application.class, args);
      }
  }
  {% endhighlight %}
  * It also takes care of the binding of the `Servlet`, `Filter`, and `ServletContextInitializer` beans from the application context to the embedded servlet container.
* **Spring Web MVC**
  * MVC is known as an architectural pattern, which embodies three parts Model, View and Controller, or to be more exact it divides the application into three logical parts: the model part, the view and the controller.
  * It was used for desktop graphical user interfaces but nowadays is used in designing mobile apps and web apps.
    * The model is responsible for managing the data of the application. It receives user input from the controller.
    * The view renders presentation of the model in a particular format.
    * The controller responds to the user input and performs interactions on the data model objects. The controller receives the input, optionally validates it and then passes the input to the model.
  * The Spring Web model-view-controller (MVC) framework is designed around a DispatcherServlet that dispatches requests to handlers, with configurable handler mappings, view resolution, locale and theme resolution as well as support for uploading files.
  * The default handler is based on the `@Controller` and `@RequestMapping` annotations, offering a wide range of flexible handling methods.
  * Spring's web MVC framework is, like many other web MVC frameworks, request-driven, designed around a central servlet that dispatches requests to controllers and offers other functionality that facilitates the development of web applications.
  * Spring's `DispatcherServlet` however, does more than just that. It is completely integrated with the Spring IoC container and as such allows you to use every other feature that Spring has.
  * The `DispatcherServlet` is an expression of the “Front Controller” design pattern. A front controller is defined as a controller that handles all requests for a Web Application. `DispatcherServlet` servlet is the front controller in Spring MVC that intercepts every request and then dispatches requests to an appropriate controller.
    ![high-level-spring-mvc-flow]({{ site.url }}/images/high-level-spring-mvc-flow.png)
* **JPA**
* **Spring Data**
### Beans
* Beans are used when you need a long lived instance (mostly the lifetime of the service) of an object.
* Beans are singleton. Beans have a single instance of an object running, and keep getting reused.
* Dependent classes use *Dependency Injection* to access the instance of the bean. (eg using `@Autowired`).
* Beans can be created using many annotations. eg `@Bean`, `@Component`, `@Service`, `@Controller` etc.
{% highlight java linenos %}
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
{% endhighlight %}
* `@Component` (and `@Service` and `@Repository`) are used to auto-detect and auto-configure beans using classpath scanning. There's an implicit one-to-one mapping between the annotated class and the bean (i.e. one bean per class). Control of wiring is quite limited with this approach, since it's purely declarative.
* `@Bean` is used to explicitly declare a single bean, rather than letting Spring do it automatically as above. It decouples the **declaration** of the bean from the class **definition**, and lets you create and configure beans exactly how you choose.
* Let's imagine that you want to wire components from 3rd-party libraries (you don't have the source code so you can't annotate its classes with `@Component`), so automatic configuration is not possible. The `@Bean` annotation returns an object that spring should register as bean in application context. The body of the method bears the logic responsible for creating the instance.
* The `@Repository` annotation is a marker for any class that fulfils the role or stereotype of a repository (also known as Data Access Object or DAO)
* The `@Controller` annotation indicates that a particular class serves the role of a controller. The dispatcher scans the classes annotated with @Controller and detects methods annotated with `@RequestMapping` annotations within them. We can use `@RequestMapping` on/in only those methods whose classes are annotated with @Controller and it will NOT work with `@Component`, ``@Service`, `@Repository`.
* `@Service` beans hold the business logic and call methods in the repository layer. Apart from the fact that it's used to indicate, that it's holding the business logic, there’s nothing else noticeable in this annotation.
### Swagger-SDK
* API as first class objects:
* codegen can be used to create yaml to model, and the `swagger-sdk` model classes are used as dependency in server project.
* the codegen is generated from `server/**/resources/public/*.yaml`, ie the public APIs defined for the service. (eg `DoSomethingRequest`: yaml object that defines the request body for `doSomething`)
* swagger-sdk can also has `src/java/**/*.java` files to add more supporting classes, generally used for exposing internal classes that are not needed in API documentation, but maybe needed for using the SDK (for example, model files).
* Both yaml (public class) and internal java files (internal class) are avaible in SDK. The SDK is used by `./server` and `./server`.
* `api-doc-external.yaml` is the external API endpoint. It is used to create the documentation for REST APIs and also generates "client" classes and function to use these API through an SDK.
{% highlight yaml linenos %}
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
{% endhighlight %}
gets converted into something like:
{% highlight java linenos %}
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
{% endhighlight %}
* So, finally the SDK has 3 main components:
  * `com.vmware.hercules.foobar.client.api.FooBarServiceApi`: These are the client APIs in the SDK that can be used to call the REST endpoint programatically.
  * `com.vmware.hercules.foobar.model`
    * Internal classes that were present in `swagger-sdk/**/src`
    * External classes generated from API defs yaml files in `server/**/resources/public`
### Marshalling/Unmarshalling Java Objects
* The SDKs published by the service help the clients, using `RestTemplate`, perform REST operations. These REST operations return json strings.
* These strings are actually objects in string form, that need to be deserialized into java objects.
* RestTemplate has a mechanism of doing this. `RestTemplate::getForObject(url, class)` can be used to perform a get operation and serialize the output to a class.
* The server and the client need to share this class. They do this by either:
  * Copying the implementation of this class from server to the client (duplicate but independent code).
  * server publishes the SDK using `swagger-sdk` with these classes present in `model`