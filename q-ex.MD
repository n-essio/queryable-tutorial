# 'Qeex' - Quarkus web application Error EXtension 

Quarkus extension to generate Rest Exception in declative way. Stringly inspired by:
- JBoss logging tools about the use of annotations inside a well knowed interface
-  QuarkusWebException that extendsjavax.ws.rs.WebApplicationException
- javax.ws.rs.ext.ExceptionMapper to generate custom excewption 

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
    public String projectName;    // (1)
    public int code;              // (2)
    public int httpCode;          // (3)
    public String message;        // (4)
    public String language;       // (5)

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

    /**
     * Indicates that this message has no ID.
     */
    int NONE = 0;
    /**
     * Indicates that this message should inherit the ID from another message with the same name.
     */
    int INHERIT = -1;

    /**
     * The message ID number.  Only one message with a given name may specify an ID other than {@link #INHERIT}.
     *
     * @return the message ID number
     */
    int id() default INHERIT;

    /**
     * The default format string of this message.
     * <p>
     * Expressions in the form of {@code ${property.key:default-value}} can be used for the value. If the property key is
     * prefixed with {@code sys.} a {@linkplain System#getProperty(String) system property} will be used. If the key is
     * prefixed with {@code env.} an {@linkplain System#getenv(String) environment variable} will be used. In all other cases
     * the {@code org.jboss.logging.tools.expressionProperties} processor argument is used to specify the path the properties
     * file which contains the values for the expressions.
     * </p>
     *
     * @return the format string
     */
    String value();

    /**
     * The format type of this method (defaults to {@link Format#PRINTF}).
     *
     * @return the format type
     */
    Format format() default Format.PRINTF;

    /**
     * The possible format types.
     */
    enum Format {

        /**
         * A {@link java.util.Formatter}-type format string.
         */
        PRINTF,
        /**
         * A {@link java.text.MessageFormat}-type format string.
         */
        MESSAGE_FORMAT,

        /**
         * Indicates the message should not be formatted.
         */
        NO_FORMAT,
    }

}

```



### Annotations use cases

### Application properties declarations





