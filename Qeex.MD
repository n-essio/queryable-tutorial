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


`````
## Usage

We can define our exceptions messages in 2 way:
- using java annotations inside our code
- using annotations and defining properties in the quarkus application.properties

### Annotations use cases

In our proiject, we will create a class annotated with @ExceptionBundle Annotation ( the extension, at runtime, will generate a concrete "AllExceptionsImplementation" with all methods annotated:

```java
import io.quarkus.qeex.api.annotations.QeexExceptionBundle;
import io.quarkus.qeex.api.annotations.QeexMessage;
import io.quarkus.qeex.api.exceptions.QeexWebException;

@QeexExceptionBundle(project = "FLW", language = "it")
public interface ExceptionBundle {

    @QeexMessage(message = "prova")
    QeexWebException simpleException();

    @QeexMessage(id = 102, code = 400, message = " complete e ricmplete")
    QeexWebException completeException();

    @QeexMessage(id = 103, code = 500, message = "non sovrascitto")
    QeexWebException notOverrideException();

    @QeexMessage(id = 104, code = 500, message = "one: %s - two: %s")
    QeexWebException withArguments(String one, String two);

}
```
We will @Inject in our Rest Services methods:

```java

import org.acme.quex.exception.ExceptionBundle;

import javax.inject.Inject;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Path("/hello")
public class GreetingResource {

    @Inject
    ExceptionBundle exceptionBundle;

    @GET
    @Path("/1")
    @Produces(MediaType.APPLICATION_JSON)
    public String hello() throws Exception {
        throw exceptionBundle.simpleException();
    }
    
```
The client will receive with HttpCode 500, and a message as:

```json
    {
        "projectName": "FLW",
        "id": 0,
        "code": 400,
        "message": "non va ancora 101",
        "language": "it"
    }
```

### Application properties declarations

To use the java methods:

```java

import io.quarkus.qeex.api.annotations.QeexExceptionBundle;
import io.quarkus.qeex.api.annotations.QeexMessage;
import io.quarkus.qeex.api.exceptions.QeexWebException;

@QeexExceptionBundle(project = "FLW", language = "it")
public interface ExceptionBundle {

    @QeexMessage(message = "prova")
    QeexWebException simpleException();

    @QeexMessage(id = 102, code = 400, message = " complete e ricmplete")
    QeexWebException completeException();

    @QeexMessage(id = 103, code = 500, message = "non sovrascitto")
    QeexWebException notOverrideException();

    @QeexMessage(id = 104, code = 500, message = "one: %s - two: %s")
    QeexWebException withArguments(String one, String two);

}
```

we ,ust define all error messages in application properties:

```properties

# global configuration
qeex.project=FLW
qeex.default.id=100
qeex.default.code=400
qeex.default.language=it
qeex.default.message=default message for FLW project

#override of 101 execption configuration
qeex.messages[0].id=101
qeex.messages[0].code=400
qeex.messages[0].message=non va ancora 101
qeex.messages[0].language.en=is not ok 101
qeex.messages[0].language.fr=c'esta pas ok 101

#override of 102 execption configuration
qeex.messages[1].id=102
qeex.messages[1].code=500
qeex.messages[1].message=non va ancora 102
qeex.messages[1].language.en=is not ok again 101
qeex.messages[1].language.fr=c'esta pas ancore ok 101

```

## Notes

- https://access.redhat.com/documentation/en-us/red_hat_jboss_enterprise_application_platform/6.4/html/development_guide/sect-jboss_logging_tools
- https://javaee.github.io/javaee-spec/javadocs/javax/ws/rs/ext/ExceptionMapper.html
- https://github.com/mkouba/gizmo-insights
- https://github.com/ikatyang/emoji-cheat-sheet/blob/master/README.md#alphanum


:green_circle: :white_circle: :red_circle:
:green_circle: :white_circle: :red_circle:
:green_circle: :white_circle: :red_circle:
