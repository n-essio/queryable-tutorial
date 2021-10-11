# 'Getting Started' con Quarkus & Queryable
Un tutorial in italiano per mostrare come creare dei microservizi con QUARKUS utilizzando Queryable Maven Plugin.

<img src="https://github.com/n-essio/queryable/blob/main/docs/queryable.png">

## Getting Started
Immaginiamo di voler gestire 3 tabelle nel nostro database Postgresql, che verranno mappate con 3 entities e sarannno interrogabili via rest tramite 3 servizi rest.

I 3 entities:

- Team (uuid, name, tags)
- Developer (uuid, name, surname, team_uuid, active)
- Project (uuid, name, budget, developers_uuid)

Che saranno interrogabili (in GET, POST, PUT, DELETE), ai 3 indirizzi:

- http://localhost:8080/api/teams
- http://localhost:8080/api/developers
- http://localhost:8080/api/projects

### Prerequisiti

Cominciamo con la creazione di un progetto Quarkus, con le estensioni per Hibernate Panache, Resteasy e il ns Queryable. 
Comandi maven da eseguire in sequenza

```
mvn io.quarkus.platform:quarkus-maven-plugin:2.3.0.Final:create \
        -DprojectGroupId=it.queryable \
        -DprojectArtifactId=myteam \
        -DclassName="it.queryable.myteam.service.rs.GreetingResource" \
        -Dpath="/myteam"
cd myteam

```

Installiamo le estensioni Quarkus:

```
./mvnw quarkus:add-extension -Dextensions="jdbc-postgresql,resteasy-jackson,hibernate-orm-panache"
```

Aggiungiamo la dipendenza Queryable al progetto ed installiamo le api rest:

```
./mvnw it.n-ess.queryable:queryable-maven-plugin:1.0.11:add
./mvnw queryable:install
```
### Siamo pronti per definire i nostri JPA Entities.

#### Team

```
package it.queryable.myteam.model;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import it.ness.queryable.annotations.*;

import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Table;

import static it.queryable.myteam.management.AppConstants.TEAMS_PATH;

@Entity
@Table(name = "teams")
@QRs(TEAMS_PATH)
@QOrderBy("name asc")
public class Team extends PanacheEntityBase {

    @GeneratedValue(generator = "uuid")
    @GenericGenerator(name = "uuid", strategy = "uuid2")
    @Column(name = "uuid", unique = true)
    @Id
    @Q
    @QList
    public String uuid;

    @QLike
    public String name;

    @QLikeList
    public String tags;
}
```
#### Developer

```
package it.queryable.myteam.model;

import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import it.ness.queryable.annotations.*;

import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Table;

import static it.queryable.myteam.management.AppConstants.DEVELOPERS_PATH;

//    Developer (uuid, name, surname, team_uuid, active)


@Entity
@Table(name = "developers")
@QRs(DEVELOPERS_PATH)
@QOrderBy("surname asc")
public class Developer extends PanacheEntityBase {

    @GeneratedValue(generator = "uuid")
    @GenericGenerator(name = "uuid", strategy = "uuid2")
    @Column(name = "uuid", unique = true)
    @Id
    @Q
    @QList
    public String uuid;

    @QLike
    public String name;

    @QLike
    public String surname;

    @QList
    public String team_uuid;

    @QLogicalDelete
    boolean active = true;
}
```
#### Project

```
package it.queryable.myteam.model;


import io.quarkus.hibernate.orm.panache.PanacheEntityBase;
import it.ness.queryable.annotations.*;

import javax.persistence.*;

import java.math.BigDecimal;
import java.util.List;

import static it.queryable.myteam.management.AppConstants.PROJECTS_PATH;

// - Project (uuid, name, budget, developers_uuid)

@Entity
@Table(name = "projects")
@QRs(PROJECTS_PATH)
@QOrderBy("name asc")
public class Project extends PanacheEntityBase {

    @GeneratedValue(generator = "uuid")
    @GenericGenerator(name = "uuid", strategy = "uuid2")
    @Column(name = "uuid", unique = true)
    @Id
    @Q
    @QList
    public String uuid;

    @QLike
    public String name;

    @Q
    public BigDecimal budget;

    @ElementCollection(fetch = FetchType.LAZY)
    public List<String> developers_uuid;

}
```

Le rotte dei nostri microservizi le definiamo qui:

```
package it.queryable.myteam.management;

public class AppConstants {
public static final String API_PATH = "/api";
public static final String DEVELOPERS_PATH = API_PATH + "/developers";
public static final String GREETING_PATH = API_PATH + "/greetings";
public static final String PROJECTS_PATH = API_PATH + "/projects";
public static final String TEAMS_PATH = API_PATH + "/teams";
}
```


### Per generare le classi REST ed i filtri Hibernate nei nostri entities, lanciamo il comando:

```
./mvnw queryable:source
```

#### Un esempio di filtri generati:

```
@Entity
@Table(name = "teams")
@QRs(TEAMS_PATH)
@QOrderBy("name asc")
@FilterDef(name = "like.tagses", parameters = @ParamDef(name = "tagses", type = "string"))
@Filter(name = "like.tagses", condition = "lower(tagses) LIKE :tagses")
@FilterDef(name = "obj.uuid", parameters = @ParamDef(name = "uuid", type = "string"))
@Filter(name = "obj.uuid", condition = "uuid = :uuid")
@FilterDef(name = "obj.uuids", parameters = @ParamDef(name = "uuids", type = "string"))
@Filter(name = "obj.uuids", condition = "uuid IN (:uuids)")
@FilterDef(name = "like.name", parameters = @ParamDef(name = "name", type = "string"))
@Filter(name = "like.name", condition = "lower(name) LIKE :name")
public class Team extends PanacheEntityBase {
```

#### E nella classe REST:

```
@Path(TEAMS_PATH)
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
@Singleton
public class TeamServiceRs extends RsRepositoryServiceV3<Team, String> {

	public TeamServiceRs() {
		super(Team.class);
	}

	@Override
	protected String getDefaultOrderBy() {
		return "name asc";
	}

	@Override
	public PanacheQuery<Team> getSearch(String orderBy) throws Exception {
		String query = null;
		Map<String, Object> params = null;
		if (nn("like.tagses")) {
			String[] tagses = get("like.tagses").split(",");
			StringBuilder sb = new StringBuilder();
			if (null == params) {
				params = new HashMap<>();
			}
			for (int i = 0; i < tagses.length; i++) {
				final String paramName = String.format("tags%d", i);
				sb.append(String.format("tags LIKE :%s", paramName));
				params.put(paramName, tagses[i]);
				if (i < tagses.length - 1) {
					sb.append(" OR ");
				}
			}
			if (null == query) {
				query = sb.toString();
			} else {
				query = query + " OR " + sb.toString();
			}
		}
		PanacheQuery<Team> search;
		Sort sort = sort(orderBy);
		if (sort != null) {
			search = Team.find(query, sort, params);
		} else {
			search = Team.find(query, params);
		}
		if (nn("obj.uuid")) {
			search.filter("obj.uuid", Parameters.with("uuid", get("obj.uuid")));
		}
		if (nn("obj.uuids")) {
			String[] uuids = get("obj.uuids").split(",");
			getEntityManager().unwrap(Session.class).enableFilter("obj.uuids").setParameterList("uuids", uuids);
		}
		if (nn("like.name")) {
			search.filter("like.name", Parameters.with("name", likeParamToLowerCase("like.name")));
		}
		return search;
	}
}
```


A questo punto verranno generate le classi REST nel package it/queryable/myteam/service/rs/: 

- DeveloperServiceRs
- GreetingResource
- GreetingServiceRs
- ProjectServiceRs
- TeamServiceRs

Ed all'interno degli entities compariranno i filtri hibernate.

Per testare il progetto, dobbiamo aggiungere la configurazione (in application.properties) al ns db di test - lanciatro utilizzando docker-compose:

```
quarkus.datasource.db-kind=postgresql
quarkus.datasource.username=myteam
quarkus.datasource.password=myteam
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/myteam
quarkus.hibernate-orm.database.generation=update
```

#### Nella cartella  /docker folder aggiungeremo il file docker-compose.yml :

```
version: '2'

services:
  postgresql:
    container_name: myteam
    image: postgres:11.8-alpine
    environment:
      POSTGRES_PASSWORD: myteam
      POSTGRES_USER: myteam
      POSTGRES_DB: myteam
    ports:
      - '5432:5432'
  pgadmin4:
    container_name: nso_orders_pgadmin4
    image: dpage/pgadmin4
    ports:
      - '5050:5050'
      - '85:80'
    links:
      - postgresql:postgresql
    depends_on:
      - postgresql
    environment:
      PGADMIN_DEFAULT_EMAIL: myteam@n-ess.it
      PGADMIN_DEFAULT_PASSWORD: myteam

```

#### Possiamo quindi lanciare il nostro database:

```
 docker-compose -f docker/docker-compose.yml down
 docker-compose -f docker/docker-compose.yml up -d
```

Ed infine:
```
 mvn compile quarkus:dev
```
#### Vedremo comparire nella shell, i seguenti logs:

```
[INFO] Scanning for projects...
[INFO]
[INFO] ------------------------< it.queryable:myteam >-------------------------
[INFO] Building myteam 1.0.0-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO]
[INFO] --- quarkus-maven-plugin:1.12.1.Final:generate-code (default) @ myteam ---
[INFO]
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ myteam ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 2 resources
[INFO]
[INFO] --- maven-compiler-plugin:3.8.1:compile (default-compile) @ myteam ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- quarkus-maven-plugin:1.12.1.Final:dev (default-cli) @ myteam ---
Listening for transport dt_socket at address: 5005
__  ____  __  _____   ___  __ ____  ______
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/
2021-03-04 00:52:39,721 INFO  [io.quarkus] (Quarkus Main Thread) myteam 1.0.0-SNAPSHOT on JVM (powered by Quarkus 1.12.1.Final) started in 2.894s. Listening on: http://localhost:8080
2021-03-04 00:52:39,723 INFO  [io.quarkus] (Quarkus Main Thread) Profile dev activated. Live Coding activated.
2021-03-04 00:52:39,723 INFO  [io.quarkus] (Quarkus Main Thread) Installed features: [agroal, cdi, hibernate-orm, hibernate-orm-panache, jdbc-postgresql, mutiny, narayana-jta, resteasy, resteasy-jackson, smallrye-context-propagation]
```

La lista degli endpoint (visibile scrivendo un path inesistente: http://localhost:8080/xxx

<img src="endpoint.png">

## Proviamo a generare alcuni TEAMS:

```
curl --location --request POST 'http://localhost:8080/api/teams' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "primo",
    "tags": "java,angular"
}'

curl --location --request POST 'http://localhost:8080/api/teams' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "secondo",
    "tags": "typescrypt,react"
}'

curl --location --request POST 'http://localhost:8080/api/teams' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "terzo",
    "tags": "scala,react"
}'
```

e vedremo nella SHELL:

```
2021-03-04 01:08:27,655 INFO  [it.que.myt.ser.rs.TeamServiceRs_Subclass] (executor-thread-1) persist
2021-03-04 01:08:27,772 INFO  [it.que.api.fil.CorsFilter] (executor-thread-1) POST - /api/teams
2021-03-04 01:10:20,104 INFO  [it.que.myt.ser.rs.TeamServiceRs_Subclass] (executor-thread-1) persist
2021-03-04 01:10:20,109 INFO  [it.que.api.fil.CorsFilter] (executor-thread-1) POST - /api/teams
2021-03-04 01:10:27,779 INFO  [it.que.myt.ser.rs.TeamServiceRs_Subclass] (executor-thread-1) persist
2021-03-04 01:10:27,787 INFO  [it.que.api.fil.CorsFilter] (executor-thread-1) POST - /api/teams
```

#### Proviamo a generare alcuni DEVELOPERS:

```
curl --location --request POST 'http://localhost:8080/api/developers' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "fiorenzo",
    "surname": "pizza",
    "team_uuid": "081fa2b1-fffc-4797-81fc-cfb54a866fcf"
}'

curl --location --request POST 'http://localhost:8080/api/developers' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "andrea",
    "surname": "rossi",
    "team_uuid": "081fa2b1-fffc-4797-81fc-cfb54a866fcf"
}'

curl --location --request POST 'http://localhost:8080/api/developers' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "giovanni",
    "surname": "rana",
    "team_uuid": "081fa2b1-fffc-4797-81fc-cfb54a866fcf"
}'
```
e vedremo nella SHELL:
```
2021-03-04 01:16:39,492 INFO  [it.que.api.fil.CorsFilter] (executor-thread-1) POST - /api/developers
2021-03-04 01:17:10,242 INFO  [it.que.myt.ser.rs.DeveloperServiceRs_Subclass] (executor-thread-1) persist
2021-03-04 01:17:10,248 INFO  [it.que.api.fil.CorsFilter] (executor-thread-1) POST - /api/developers
2021-03-04 01:17:34,262 INFO  [it.que.myt.ser.rs.DeveloperServiceRs_Subclass] (executor-thread-1) persist
2021-03-04 01:17:34,267 INFO  [it.que.api.fil.CorsFilter] (executor-thread-1) POST - /api/developers
```

#### A questo punto, qualche query REST:

```
http://localhost:8080/api/developers/5915fe2b-bb95-440a-b895-889de7f8808c
http://localhost:8080/api/developers?like.surname=fi
http://localhost:8080/api/developers?obj.team_uuid=081fa2b1-fffc-4797-81fc-cfb54a866fcf

```

Buon divertimento con **QUARKUS & QUERYABLE**.

##  Qualche link utile:
- https://quarkus.io/guides/
- https://quarkus.io/guides/rest-json
- https://quarkus.io/guides/hibernate-orm-panache
- https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#pc-filter
- https://github.com/n-essio/queryable


## About @FilterDef types

The declared types:
- https://docs.jboss.org/hibernate/stable/core.old/reference/en/html/mapping-types.html


### 5.2.2. Basic value types
The built-in basic mapping types may be roughly categorized into

integer, long, short, float, double, character, byte, boolean, yes_no, true_false
Type mappings from Java primitives or wrapper classes to appropriate (vendor-specific) SQL column types. boolean, yes_no and true_false are all alternative encodings for a Java boolean or java.lang.Boolean.

####  string
A type mapping from java.lang.String to VARCHAR (or Oracle VARCHAR2).

#### date, time, timestamp
Type mappings from java.util.Date and its subclasses to SQL types DATE, TIME and TIMESTAMP (or equivalent).

####  calendar, calendar_date
Type mappings from java.util.Calendar to SQL types TIMESTAMP and DATE (or equivalent).

####  big_decimal, big_integer
Type mappings from java.math.BigDecimal and java.math.BigInteger to NUMERIC (or Oracle NUMBER).

####  locale, timezone, currency
Type mappings from java.util.Locale, java.util.TimeZone and java.util.Currency to VARCHAR (or Oracle VARCHAR2). Instances of Locale and Currency are mapped to their ISO codes. Instances of TimeZone are mapped to their ID.

####  class
A type mapping from java.lang.Class to VARCHAR (or Oracle VARCHAR2). A Class is mapped to its fully qualified name.

####  binary
Maps byte arrays to an appropriate SQL binary type.

####  text
Maps long Java strings to a SQL CLOB or TEXT type.

####  serializable
Maps serializable Java types to an appropriate SQL binary type. You may also indicate the Hibernate type serializable with the name of a serializable Java class or interface that does not default to a basic type.

####  clob, blob
Type mappings for the JDBC classes java.sql.Clob and java.sql.Blob. These types may be inconvenient for some applications, since the blob or clob object may not be reused outside of a transaction. (Furthermore, driver support is patchy and inconsistent.)

####  imm_date, imm_time, imm_timestamp, imm_calendar, imm_calendar_date, imm_serializable, imm_binary
Type mappings for what are usually considered mutable Java types, where Hibernate makes certain optimizations appropriate only for immutable Java types, and the application treats the object as immutable. For example, you should not call Date.setTime() for an instance mapped as imm_timestamp. To change the value of the property, and have that change made persistent, the application must assign a new (nonidentical) object to the property.

Unique identifiers of entities and collections may be of any basic type except binary, blob and clob. (Composite identifiers are also allowed, see below.)

The basic value types have corresponding Type constants defined on org.hibernate.Hibernate. For example, Hibernate.STRING represents the string type.

## To test in java before use parameters

```java 
private void verifyParameterInFilter(String filterName, String name, Object value) throws Exception {
    FilterDefinition definition = getEntityManager().unwrap(Session.class).getSessionFactory()
            .getFilterDefinition(filterName);
    Type type = definition.getParameterType(name);
    if (type == null) {
        throw new IllegalArgumentException("Undefined filter parameter [" + name + "]");
    }
    logger.info("************************************");
    logger.info("************************************");
    logger.info("filterName:" + filterName);
    logger.info("name:" + name);
    logger.info("value:" + value);
    logger.info("filter def type: " + type.getName());
    logger.info("value type: " + value.getClass());
    if (value != null && !type.getReturnedClass().isAssignableFrom(value.getClass())) {
        throw new IllegalArgumentException("Incorrect type for parameter [" + name + "]");
    }
    logger.info("************************************");
    logger.info("************************************");
}
```
