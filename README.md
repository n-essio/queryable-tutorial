# queryable-tutorial
Un tutorial 

## Getting Started
Abbiamo 3 tabelle su db, 3 entities, 3 servizi rest:

- Team (uuid, name, tags)
- Developer (uuid, name, surname, team_uuid, active)
- Project (uuid, name, budget, developers_uuid)


### Prerequisites

Cominciamo con la creazione di un progetto Quarkus, con le estensioni per Hibernate Panache, Resteasy e il ns Queryable. 
Comandi maven da eseguire in sequenza

```
mvn io.quarkus:quarkus-maven-plugin:1.12.0.Final:create \
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

Aggiungiamo Queryable e inizializziamo le api rest:

```
./mvnw it.n-ess.queryable:queryable-maven-plugin:1.0.6:add-querable
./mvnw queryable:install
```
Siamo pronti per definire i nostri JPA Entities:
```
./mvnw mvn it.n-ess.queryable:queryable-maven-plugin:1.0.6:add
./mvnw queryable:install
```
Team entity:

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
    boolean active;
}
```


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

And then:

```
./mvnw queryable:source
```

To test, we need to configure a database and all properties (in application.properties):

```
quarkus.datasource.db-kind=postgresql
quarkus.datasource.username=myteam
quarkus.datasource.password=myteam
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/myteam
quarkus.hibernate-orm.database.generation=update
```

We will use a docker-compose.yml (in /docker folder):

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

To start the database:

```
 docker-compose -f docker/docker-compose.yml down
 docker-compose -f docker/docker-compose.yml up
```

Ed infine:
```
 mvn   compile quarkus:dev
```


