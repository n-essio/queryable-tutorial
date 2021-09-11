# 'Qeex' - Quarkus web application Error EXtension 

Quarkus extension to generate Rest Exception in declative way. Stringly inspired by:
- JBoss logging tools about the use of annotations inside a well knowed interface
-  QuarkusWebException that extendsjavax.ws.rs.WebApplicationException
- javax.ws.rs.ext.ExceptionMapper to generate custom excewption 

## Getting Started

Immagine that you defined a custom Exception as:

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

```
public class QuarkusWebException extends Exception {
    public String projectName;    // (1)
    public int code;              // (2)
    public int httpCode;          // (3)
    public String message;        // (4)
    public String language;        // (4)

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



In JaxRS you can define

@Provider
public class QuarkusWebExceptionProvider implements ExceptionMapper<QuarkusWebException> {

  @Override
  public Response toResponse(QuarkusWebException e) {
    e.printStackTrace();
    return Response.ok(e.message).build();
  }
}

### Annotations use cases

### Application properties declarations





