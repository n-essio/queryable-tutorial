# queryable-tutorial
Un tutorial 

## Getting Started
Abbiamo 3 tabelle su db, 3 entities, 3 servizi rest.


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
./mvnw mvn it.n-ess.queryable:queryable-maven-plugin:1.0.6:add
./mvnw queryable:install
```
