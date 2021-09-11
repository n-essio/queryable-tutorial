# "Qeex"- Quarkus web application Error EXtension


Quarkus extension to generate Rest Exception in declative way. Strongly inspired by:
- JBoss logging tools about the use of annotations inside a well knowed interface
- T extends javax.ws.rs.WebApplicationException
- javax.ws.rs.ext.ExceptionMapper to generate custom rest Exception 

## What the extension offers by design

A custom ExceptionMessage that contains some utils fields to client:

```
public class ExceptionMessage {
    public String projectName;
    public int code;
    public String message;

    public ExceptionMessage(String projectName, int code, String message) {
        this.projectName = projectName;
        this.code = code;
        this.message = message;
    }
}
```
That message will be generated dinamically using a custom Exception:

```
public class QuarkusWebException extends Exception {
    public String projectName;    // (:one:)
    public int code;              // (:two:)
    public int httpCode;          // (:three:)
    public String message;        // (:four:)

    public QuarkusWebException(String projectName,
               int code,
               int httpCode,
               String message) {
        super();
        this.projectName = projectName;
        this.code = code;
        this.httpCode = httpCode;
        this.message = message;
    }
    
    public ExceptionMessage msg() {
        return new ExceptionMessage(this.projectName, this.code, this.message, this.language);
    }


}
```

A custom ExceptionMapper that will generates Rest Response with inside out Messages:

```
import javax.ws.rs.core.Context;
import javax.ws.rs.core.Response;
import javax.ws.rs.core.UriInfo;
import javax.ws.rs.ext.ExceptionMapper;
import javax.ws.rs.ext.Provider;

@Provider
public class QuarkusWebExceptionProvider implements ExceptionMapper<QuarkusWebException> {

    @Context
    UriInfo uriInfo;

    @Override
    public Response toResponse(QuarkusWebException exception) {
        exception.printStackTrace();
        String language = uriInfo.getQueryParameters().getFirst("language");
        return Response
                .status(exception.httpCode)
                .entity(exception.msg())
                .build();
    }
}
```
In "Jboss Annotation tool" style, the extension define two custom annotations @ExceptionBundle and @Message:

```
import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

import static java.lang.annotation.ElementType.TYPE;
import static java.lang.annotation.RetentionPolicy.RUNTIME;


@Target({ TYPE })
@Retention(RUNTIME)
@Documented
public @interface ExceptionBundle {

    String projectCode();
    int id() default 100;
    int code() default 402;
}

```

We need also to define:

```
import static java.lang.annotation.ElementType.METHOD;
import static java.lang.annotation.RetentionPolicy.CLASS;

import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

@Target(METHOD)
@Retention(CLASS)
@Documented
public @interface Message {

    int id() default -1;
    String message();
    int code() default 402;

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
    <artifactId>quarkus-qex</artifactId>
    <version>${latest.release.version}</version>
</dependency>
```

## Usage

We can define our exceptions messages in 2 way:
- using java annotations inside our code
- using annotations and defining properties in the quarkus application.properties

### Annotations use cases

In our proiject, we will create a class annotated with @ExceptionBundle Annotation ( the extension, at runtime, will generate a concrete "AllExceptionsImplementation" with all methods annotated:

```

@ExceptionBundle(projectCode = "QEEX")
public interface AllExceptions {

    @Message(id= 101, message = "simpe exception for client", code=402)
    QuarkusWebException simpleException();
    
    @Message(id= 102, message = "simpe exception for client", code=402)
    QuarkusWebException complexException();

}
```
We will @Inject in our Rest Services methods:

```

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
    public String hello() throws QEx {
        throw allExceptions.simpleException();
    }
}
    
```
The client will receive with HttpCode 402, and a message as:

    {
        "projectName": "QEEX",
        "code": 101,
        "message": "simpe exception for client"                                                        
    }


### Application properties declarations

In alternative we can define messages inside quarkus application.properties :


```
import it.ness.qeex.annotations.ExceptionBundle;
import it.ness.qeex.annotations.Message;
import it.ness.qeex.annotations.QuarkusWebException;

@ExceptionBundle(projectCode = "FLW")
public interface AllExceptions {

    @Message(code= 101)
    QuarkusWebException simpleException();
    
    @Message(code= 102)
    QuarkusWebException simpleException102();
    
    @Message(code= 103, httpCode=405, message="all inside java code")
    QuarkusWebException simpleException103();

}
```

and we can define all error messages in application properties:

```
quarkus.qeex.projectCode=FLW
quarkus.qeex.default.id=100
quarkus.qeex.default.code=500

quarkus.qeex.exceptions.101.message=semplice eccezione
quarkus.qeex.exceptions.101.httpCode=402

quarkus.qeex.exceptions.102.message=semplice eccezione 102
quarkus.qeex.exceptions.103.message=sovrascrivo 103
```

## Notes

- https://access.redhat.com/documentation/en-us/red_hat_jboss_enterprise_application_platform/6.4/html/development_guide/sect-jboss_logging_tools
- https://javaee.github.io/javaee-spec/javadocs/javax/ws/rs/ext/ExceptionMapper.html
- https://github.com/mkouba/gizmo-insights
- https://github.com/ikatyang/emoji-cheat-sheet/blob/master/README.md#alphanum


:green_circle: :white_circle: :red_circle:
:green_circle: :white_circle: :red_circle:
:green_circle: :white_circle: :red_circle:
