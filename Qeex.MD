# "Qeex" - Quarkus web application Error EXtension  :green_circle: :white_circle: :red_circle:


Quarkus extension to generate Rest Exception in declative way. Strongly inspired by:
- JBoss logging tools about the use of annotations inside a well knowed interface
- T extends javax.ws.rs.WebApplicationException
- javax.ws.rs.ext.ExceptionMapper to generate custom rest Exception 

## Getting Started

Immagine that you defined a custom ExceptionMessage that contains some utils fields to client:

```
public class ExceptionMessage {
    public String projectName;
    public int code;
    public String message;
    public String language;


    public ExceptionMessage(String projectName, int code, String message, String language) {
        this.projectName = projectName;
        this.code = code;
        this.message = message;
        this.language = language;
    }
}
```
That message will be generated dinamically using:

```
public class QuarkusWebException extends Exception {
    public String projectName;    // (:one:)
    public int code;              // (:two:)
    public int httpCode;          // (:three:)
    public String message;        // (:four:)
    public String language;       // (:five:)

    public QEx(String projectName,
               int code,
               int httpCode,
               String message,
               String languag) {
        super();
        this.projectName = projectName;
        this.code = code;
        this.httpCode = httpCode;
        this.message = message;
        this.language = language;
    }
    
    public ExceptionMessage to() {
      return new ExceptionMessage(this.projectName, this.code, this.message, this.language);
    }


}
```

In JaxRS you can define an ExceptionMapper

```
import javax.ws.rs.core.Response;
import javax.ws.rs.ext.ExceptionMapper;
import javax.ws.rs.ext.Provider;

@Provider
public class QuarkusWebExceptionProvider implements ExceptionMapper<QuarkusWebException> {

    @Override
    public Response toResponse(QuarkusWebException exception) {
        exception.printStackTrace();
        return Response
                .status(exception.httpCode)     // (1)
                .entity(exception.to())         // (2)
                .build();
    }
}
```
In style of Jboss Annottaion toool, we will define two custom annotations:

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
    int code();
}

```

We need also to define:

```
import static java.lang.annotation.ElementType.METHOD;
import static java.lang.annotation.RetentionPolicy.CLASS;

import java.lang.annotation.Documented;
import java.lang.annotation.Retention;
import java.lang.annotation.Target;

/**
 * Assigns a message string to a resource method.  The method arguments are used to supply the positional parameter
 * values for the method.
 *
 * @author <a href="mailto:david.lloyd@redhat.com">David M. Lloyd</a>
 */
@Target(METHOD)
@Retention(CLASS)
@Documented
public @interface Message {

    int id() default INHERIT;
    String value();

}

```
How to use? Follows the 2 way:
- using annotations
- using application properties

### Annotations use cases

We will create a class annotated with ExceptionBundle Annotation, that at runtime will have the Implementation

```


@ExceptionBundle(projectCode = "FLW")
public interface AllExceptions {

    @Message(code= 123, message = "simpe exception for client", httpCode=402)
    QuarkusWebException simpleException();
    
    @Message(code= 12333, message = "simpe exception for client", httpCode=402)
    QuarkusWebException simpleException();

}
```
and we will use in our project in this way:

```

    @Inject
    AllExceptions allExceptions;

    @GET
    @Produces(MediaType.TEXT_PLAIN)
    public String hello() throws QEx {
        throw allExceptions.simpleException();
    }
```
The client will receive with httpCode 402, and a message as:

{
        projectName ='quarkus';
        code = 134;
        message = 2                                                              w;
        language = IT;
}


### Application properties declarations

The alternative use is:


```

import it.ness.qex.annotations.ExceptionBundle;
import it.ness.qex.annotations.Message;
import it.ness.qex.annotations.QEx;

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
quarkus.qeex.default.code=100
quarkus.qeex.default.httpCode=500
quarkus.qeex.default.language=it

quarkus.qeex.exceptions.101.it.message=semplice eccezione
quarkus.qeex.exceptions.101.it.httpCode=402
quarkus.qeex.exceptions.101.en.message=sample exception
quarkus.qeex.exceptions.101.en.httpCode=402

quarkus.qeex.exceptions.102.it.message=semplice eccezione 102
quarkus.qeex.exceptions.102.en.message=sample exception 102

quarkus.qeex.exceptions.103.it.message=sovrascrivo 103
quarkus.qeex.exceptions.103.en.message=override 103
```

## Notes

- https://access.redhat.com/documentation/en-us/red_hat_jboss_enterprise_application_platform/6.4/html/development_guide/sect-jboss_logging_tools
- https://javaee.github.io/javaee-spec/javadocs/javax/ws/rs/ext/ExceptionMapper.html
- https://github.com/mkouba/gizmo-insights
- https://github.com/ikatyang/emoji-cheat-sheet/blob/master/README.md#alphanum



