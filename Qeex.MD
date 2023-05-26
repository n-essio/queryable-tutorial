# "Qeex"- Quarkus web application Error EXtension

This is a Quarkus extension for generate Rest Exception in declarative way.
Strongly inspired by:
- JBoss logging tools - about the use of annotations inside an annotated interface

This extension will provide:
- jax-rs ExceptionMapper that will generate custom json exception messages, ie:

HTTP STATUS: 500
RESPONSE ENTITY:
```json
{
    "project": "XXXXX",
    "id": 101,
    "code": 500,
    "message": "simpe message for client"     
    "language": "it"
}
```

TODO:
- provides support for the internationalization and localization of exception messages.

## What the extension offers by design

A custom ExceptionMessage that contains some utils fields to client:

```java
public class QeexWebException {
    public String projectName;
    public int id;
    public int code;
    public String message;
    public String language;


    protected QeexWebException() {
    }

    public static QeexWebException builder(String projectName) {
        var quex = new QeexWebException();
        quex.projectName = projectName;
        return quex;
    }

    public QeexWebException id(int id) {
        this.id = id;
        return this;
    }

    public QeexWebException code(int code) {
        this.code = code;
        return this;
    }

    public QeexWebException message(String message) {
        this.message = message;
        return this;
    }

    public QeexWebException language(String language) {
        this.language = language;
        return this;
    }


    public String toJson() {
        return "{\"projectName\":\"" + this.projectName + "\"," +
                "\"id\":" + this.id + ", " +
                "\"code\":" + this.code + ", " +
                "\"message\":\"" + this.message + "\", " +
                "\"language\":\"" + this.language + "\"" +
                "}";
    }

    @Override
    public String toString() {
        return "QuarkusWebException{" +
                "projectName='" + projectName + '\'' +
                ", id=" + id +
                ", code=" + code +
                ", message='" + message + '\'' +
                ", language='" + language + '\'' +
                '}';
    }
}

```
That Object will be translated in json:
```json
{
    "project": "XXXXX",
    "id": 101,
    "code": 400,
    "message": "simpe message for client"     
    "language": "it"                                                    
}
```
That message will be generated dinamically using the custom Exception, where:
- "id" is the uuid of message exception
- "code" is the http code for the response
- "project" is bundle name of exceptions
- "message" is the exception message

A custom ExceptionMapper that will generates Rest Response with inside out Messages:

```java
import io.quarkus.qeex.api.exceptions.QeexWebException;

import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;
import javax.ws.rs.ext.ExceptionMapper;
import javax.ws.rs.ext.Provider;

@Provider
public class QuarkusWebExceptionProvider implements ExceptionMapper<QeexWebException> {

    @Override
    @Produces(MediaType.APPLICATION_JSON)
    public Response toResponse(QeexWebException exception) {
        return Response
                .status(exception.code)
                .entity(exception.toJson())
                .build();
    }
}
```
In "Jboss Annotation tool" style, the extension define two custom annotations @ExceptionBundle and @Message:

```java
import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

import static java.lang.annotation.ElementType.TYPE;
import static java.lang.annotation.RetentionPolicy.RUNTIME;


@Target({ TYPE })
@Retention(RUNTIME)
@Documented
public @interface QeexExceptionBundle {

    String project();
    int id() default 100;
    int code() default 400;
    String language();
    String message() default "";
}

```

We need also to define:

```java
import javax.inject.Qualifier;
import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

import static java.lang.annotation.ElementType.METHOD;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

@Qualifier
@Target({METHOD})
@Retention(RUNTIME)
@Documented
public @interface QeexMessage {

    int id() default -1;

    String message() default "";

    int code() default 400;
}

```

## Coordinates
After configuring `quarkus BOM`:

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-bom</artifactId>
            <version>${insert.newest.quarkus.version.here}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

You can just configure the `quarkus-qex` extension by adding the following dependency:

```xml
<dependency>
    <groupId>io.quarkiverse.qex</groupId>
    <artifactId>quarkus-qeex</artifactId>
    <version>${latest.release.version}</version>
</dependency>
```

## Usage

We can define our exceptions messages in 2 way:
- using java annotations inside our code
- using annotations and defining properties in the quarkus application.properties

### Annotations use cases

In our proiject, we will create a class annotated with @ExceptionBundle Annotation ( the extension, at runtime, will generate a concrete "AllExceptionsImplementation" with all methods annotated:

```java

@ExceptionBundle(project = "QEEX")
public interface AllExceptions {

    @Message(id= 101, message = "simpe exception for client", code=500)
    QuarkusWebException simpleException();
    
    @Message(id= 102, message = "simpe exception for client", code=503)
    QuarkusWebException complexException();

}
```
We will @Inject in our Rest Services methods:

```java

import javax.inject.Inject;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Path("/hello")
public class GreetingResource {

    @Inject
    AllExceptions allExceptions;

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() throws Exception {
        throw allExceptions.simpleException();
    }
}
    
```
The client will receive with HttpCode 500, and a message as:

```json
    {
        "project": "QEEX",
        "id": 101,
        "message": "simpe exception for client"                                                        
    }
```

### Application properties declarations

We can define messages inside quarkus application.properties :


```java
import it.ness.qeex.annotations.ExceptionBundle;
import it.ness.qeex.annotations.Message;
import it.ness.qeex.annotations.QuarkusWebException;

@ExceptionBundle(project = "FLW")
public interface AllExceptions {

    @Message(id= 101)
    QuarkusWebException simpleException101();
    
    @Message(id= 102)
    QuarkusWebException simpleException102();
    
    @Message() 
    QuarkusWebException simpleException103();

}
```

and we can define all error messages in application properties:

```properties
quarkus.qeex.project=FLW
quarkus.qeex.default.id=100
quarkus.qeex.default.code=500

## 101 => message: semplice eccezione 101 , code: 500 (default)
quarkus.qeex.messages[0].id=101
quarkus.qeex.messages[0].message=semplice eccezione 101


## 102 => semplice eccezione 102 , code: 403
quarkus.qeex.messages[1].id=102
quarkus.qeex.messages[1].code=403

## 103 => simpleException103() => implicit id and code, message: semplice eccezione 103
quarkus.qeex.messages[2].id=103
quarkus.qeex.messages[2].message=semplice eccezione 103

```

## Notes

- https://access.redhat.com/documentation/en-us/red_hat_jboss_enterprise_application_platform/6.4/html/development_guide/sect-jboss_logging_tools
- https://javaee.github.io/javaee-spec/javadocs/javax/ws/rs/ext/ExceptionMapper.html
- https://github.com/mkouba/gizmo-insights
- https://github.com/ikatyang/emoji-cheat-sheet/blob/master/README.md#alphanum


:green_circle: :white_circle: :red_circle:
:green_circle: :white_circle: :red_circle:
:green_circle: :white_circle: :red_circle:
