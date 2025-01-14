
# Adding Persistence to Boot Application

In this lab we'll utilize Spring Boot, Spring Data, and Spring Data REST to create a fully-functional hypermedia-driven RESTful web service.  We'll then deploy it to Pivotal Cloud Foundry.  Along the way we'll take a brief look at [Flyway](https://flywaydb.org) which can help us manage updates to database schema and data.

## Create a Hypermedia-Driven RESTful Web Service with Spring Data REST (using JPA)

This application will allow us to create, read update and delete records in an [in-memory](http://www.h2database.com/html/quickstart.html) relational repository. We'll continue building upon the Spring Boot application we built out in Lab 1.  The first stereotype we will need is the domain model itself, which is `City`.

## Add the domain object - City

* Create the package `io.pivotal.domain` and in that package create the class `City`. Into that file you can paste the following source code, which represents cities based on postal codes, global coordinates, etc:

```java
package io.pivotal.domain;

@Data
@Entity
@Table(name="city")
public class City implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue
    private long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private String county;

    @Column(nullable = false)
    private String stateCode;

    @Column(nullable = false)
    private String postalCode;

    @Column
    private String latitude;

    @Column
    private String longitude;

}
```

Notice that we're using [JPA](http://docs.oracle.com/javaee/6/tutorial/doc/bnbpz.html) annotations on the class and its fields. We're also employing [Lombok](https://projectlombok.org/features/all), so we don't have to write a bunch of boilerplate code (e.g., getter and setter methods).  You'll need to use your IDE's features to add the appropriate import statements.

-> Hint: imports should start with `javax.persistence` and `lombok`

* Create the package `io.pivotal.repositories` and in that package create the interface `CityRepository`. Paste the following code and add appropriate imports:

```java
package io.pivotal.repositories;

@RepositoryRestResource(collectionResourceRel = "cities", path = "cities")
public interface CityRepository extends PagingAndSortingRepository<City, Long> {
}
```

You’ll need to use your IDE’s features to add the appropriate import statements.

-> Hint: imports should start with `org.springframework.data.rest.core.annotation` and `org.springframework.data.repository`

## Use Flyway to manage schema

* Edit **pom.xml** and add the following dependencies within the `dependencies`

```xml
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
            <version>5.2.4</version>
        </dependency>
        <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
            <version>3.3.0</version>
        </dependency>
```

* Create a new file named `V1_0__init_database.sql` underneath **cloudnative-spring-workshop/labs/my_work/cloud-native-spring/src/main/resources/db/migration**, add the following lines and save.

```sql
CREATE TABLE city (
   ID INTEGER PRIMARY KEY AUTO_INCREMENT,
   NAME VARCHAR(100) NOT NULL,
   COUNTY VARCHAR(100) NOT NULL,
   STATE_CODE VARCHAR(10) NOT NULL,
   POSTAL_CODE VARCHAR(10) NOT NULL,
   LATITUDE VARCHAR(15) NOT NULL,
   LONGITUDE VARCHAR(15) NOT NULL
);
```

Spring Boot comes with out-of-the-box [integration](https://docs.spring.io/spring-boot/docs/current/reference/html/howto-database-initialization.html#howto-execute-flyway-database-migrations-on-startup) support for [Flyway](https://flywaydb.org/documentation/plugins/springboot).  When we start the application it will execute a versioned [SQL migration](https://flywaydb.org/documentation/migrations#sql-based-migrations) that will create a new table in the database.

* Add the following lines to **cloudnative-spring-workshop/labs/my_work/cloud-native-spring/src/main/resources/application.yml**

```yml
spring:
  datasource:
    hikari:
      connection-timeout: 60000
      maximum-pool-size: 5
```
[Hikari](https://github.com/brettwooldridge/HikariCP/blob/dev/README.md) is a database connection pool implementation. We are limiting the number of database connections an individual application instance may consume.

## Run the _cloud-native-spring_ Application

* Return to the Terminal session you opened previously

* Run the application

```bash
mvn clean spring-boot:run
```
* Access the application using +curl+ or your web browser using the newly added REST repository endpoint at http://localhost:8080/cities. You'll see that the primary endpoint automatically exposes the ability to page, size, and sort the response JSON.

```bash
http :8080/cities

HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
Content-Type: application/hal+json;charset=UTF-8
Transfer-Encoding: chunked
Date: Thu, 28 Apr 2016 14:44:06 GMT

{
  "_embedded" : {
    "cities" : [ ]
  },
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/cities"
    },
    "profile" : {
      "href" : "http://localhost:8080/profile/cities"
    }
  },
  "page" : {
    "size" : 20,
    "totalElements" : 0,
    "totalPages" : 0,
    "number" : 0
  }
}
```

* To exit the application, type *Ctrl-C*.

So what have you done? Created four small classes, modified a build file, added some configuration and SQL migration scripts, resulting in a fully-functional REST microservice. The application's +DataSource+ is created automatically by Spring Boot using the in-memory database because no other +DataSource+ was detected in the project.

Next we'll import some data.

## Importing Data

* Copy the [import.sql](https://raw.githubusercontent.com/Pivotal-Field-Engineering/devops-workshop/master/labs/import.sql) file found in *cloudnative-spring-workshop/labs/* to **PCF-CN-Workshop-BR/labs/my_work/cloud-native-spring/src/main/resources/db/migration**.
* Rename the file to be `V1_1__seed_data.sql`. (This is a small subset of a larger dataset containing all of the postal codes in the United States and its territories).

* Restart the application.

```bash
mvn clean spring-boot:run
```

* Access the application again. Notice the appropriate hypermedia is included for *next*, *previous*, and *self*. You can also select pages and page size by utilizing `?size=n&page=n` on the URL string. Finally, you can sort the data utilizing `?sort=fieldName` (replace fieldName with a cities attribute).

```bash
http :8080/cities

HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
X-Application-Context: application
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Tue, 27 May 2014 19:59:58 GMT

{
  "_links" : {
    "next" : {
      "href" : "http://localhost:8080/cities?page=1&size=20"
    },
    "self" : {
      "href" : "http://localhost:8080/cities{?page,size,sort}",
      "templated" : true
    }
  },
  "_embedded" : {
    "cities" : [ {
      "name" : "HOLTSVILLE",
      "county" : "SUFFOLK",
      "stateCode" : "NY",
      "postalCode" : "00501",
      "latitude" : "+40.922326",
      "longitude" : "-072.637078",
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/cities/1"
        }
      }
    },

    // ...

    {
      "name" : "CASTANER",
      "county" : "LARES",
      "stateCode" : "PR",
      "postalCode" : "00631",
      "latitude" : "+18.269187",
      "longitude" : "-066.864993",
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/cities/20"
        }
      }
    } ]
  },
  "page" : {
    "size" : 20,
    "totalElements" : 42741,
    "totalPages" : 2138,
    "number" : 0
  }
}
```

* Try the following URL Paths with +curl+ to see how the application behaves:

http://localhost:8080/cities?size=5

http://localhost:8080/cities?size=5&page=3

http://localhost:8080/cities?sort=postalCode,desc

Next we'll add searching capabilities.

## Adding Search

* Let's add some additional finder methods to *CityRepository*:

```java
@RestResource(path = "name", rel = "name")
Page<City> findByNameIgnoreCase(@Param("q") String name, Pageable pageable);

@RestResource(path = "nameContains", rel = "nameContains")
Page<City> findByNameContainsIgnoreCase(@Param("q") String name, Pageable pageable);

@RestResource(path = "state", rel = "state")
Page<City> findByStateCodeIgnoreCase(@Param("q") String stateCode, Pageable pageable);

@RestResource(path = "postalCode", rel = "postalCode")
Page<City> findByPostalCode(@Param("q") String postalCode, Pageable pageable);

@Query(value ="select c from City c where c.stateCode = :stateCode")
Page<City> findByStateCode(@Param("stateCode") String stateCode, Pageable pageable);
```

-> Hint: imports should start with `org.springframework.data.domain`, `org.springframework.data.rest.core.annotation`, `org.springframework.data.repository.query`, and `org.springframework.data.jpa.repository`

* Run the application

```bash
mvn clean spring-boot:run
```

* Access the application again. Notice that hypermedia for a new *search* endpoint has appeared.

```bash
http :8080/cities

HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
X-Application-Context: application
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Tue, 27 May 2014 20:33:52 GMT

// prior omitted
    },
    "_links": {
        "first": {
            "href": "http://localhost:8080/cities?page=0&size=20"
        },
        "self": {
            "href": "http://localhost:8080/cities{?page,size,sort}",
            "templated": true
        },
        "next": {
            "href": "http://localhost:8080/cities?page=1&size=20"
        },
        "last": {
            "href": "http://localhost:8080/cities?page=2137&size=20"
        },
        "profile": {
            "href": "http://localhost:8080/profile/cities"
        },
        "search": {
            "href": "http://localhost:8080/cities/search"
        }
    },
    "page": {
        "size": 20,
        "totalElements": 42741,
        "totalPages": 2138,
        "number": 0
    }
}
```

* Access the new **search** endpoint:
`http://localhost:8080/cities/search`

```bash
http :8080/cities/search

HTTP/1.1 200 OK
Server: Apache-Coyote/1.1
X-Application-Context: application
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Tue, 27 May 2014 20:38:32 GMT

{
    "_links": {
        "postalCode": {
            "href": "http://localhost:8080/cities/search/postalCode{?q,page,size,sort}",
            "templated": true
        },
        "state": {
            "href": "http://localhost:8080/cities/search/state{?q,page,size,sort}",
            "templated": true
        },
        "nameContains": {
            "href": "http://localhost:8080/cities/search/nameContains{?q,page,size,sort}",
            "templated": true
        },
        "name": {
            "href": "http://localhost:8080/cities/search/name{?q,page,size,sort}",
            "templated": true
        },
        "findByStateCode": {
            "href": "http://localhost:8080/cities/search/findByStateCode{?stateCode,page,size,sort}",
            "templated": true
        },
        "self": {
            "href": "http://localhost:8080/cities/search"
        }
    }
}
```

Note that we now have new search endpoints for each of the finders that we added.

* Try a few of these endpoints in [Postman](https://www.getpostman.com). Feel free to substitute your own values for the parameters.

http://localhost:8080/cities/search/postalCode?q=01229

http://localhost:8080/cities/search/name?q=Springfield

http://localhost:8080/cities/search/nameContains?q=West&size=1

-> For further details on what's possible with Spring Data JPA, consult the [reference documentation](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#dependencies.spring-boot)

## Pushing to Cloud Foundry

* Build the application

<details>
<summary>_Gradle Help_</summary>

```
gradle build
```
</details>

<details>
<summary>_Maven Help_</summary>

```
mvn package
```
</details>

<br>

* You should already have an application manifest, **manifest.yml**, created in Lab 1; this can be reused.  You'll want to add a timeout param so that our service has enough time to initialize with its data loading:

```yaml
applications:
- name: cloud-native-spring
  random-route: true
  memory: 1024M
  instances: 1
  path: ./build/libs/cloud-native-spring-1.0-SNAPSHOT-exec.jar
  buildpacks:
  - java_buildpack_offline
  stack: cflinuxfs3
  timeout: 180 # to give time for the data to import
  env:
    JAVA_OPTS: -Djava.security.egd=file:///dev/urandom
```

* Push to Cloud Foundry:

```bash
cf push
...

Showing health and status for app cloud-native-spring in org zoo-labs / space development as cphillipson@pivotal.io...
OK

requested state: started
instances: 1/1
usage: 1G x 1 instances
urls: cloud-native-spring-apodemal-hyperboloid.cfapps.io
last uploaded: Thu Jul 28 23:29:21 UTC 2018
stack: cflinuxfs2
buildpack: java_buildpack_offline

     state     since                    cpu      memory         disk         details
#0   running   2018-07-28 04:30:22 PM   163.7%   395.7M of 1G   159M of 1G
```

* Access the application at the random route provided by CF:

`http GET https://cloud-native-spring-{random-word}.{domain}.com/cities`

**random-word** might be something like *loquacious-eagle* and **domain** might be *cfapps.io* if you happened to target Pivotal Web Services

* Let's stop the application momentarily as we prepare to swap out the database provider.

```bash
cf stop cloud-native-spring
```

## Binding to a MySQL database in Cloud Foundry

* Let's create a MySQL database instance. Hopefully, you will have [p.mysql](https://network.pivotal.io/products/pivotal-mysql) service available in CF Marketplace.

```bash
cf marketplace -s p.mysql
```
Expected output:

```bash
Getting service plan information for service p.mysql as cphillipson@pivotal.io...
OK

service plan   description                                            free or paid
db-small       This plan provides a small dedicated MySQL instance.   free
```

* Let's create an instance of `p.mysql` with `db-small` plan, e.g.

```bash
cf create-service p.mysql db-small mysql-database
```
Expected output:

```bash
Creating service instance mysql-database in org zoo-labs / space development as cphillipson@pivotal.io...
OK
```

So long as the name of the service contains `mysql` the [mysql-connector](https://dev.mysql.com/downloads/connector/j/) JDBC driver will [automatically be added](https://github.com/cloudfoundry/java-buildpack/blob/master/docs/framework-maria_db_jdbc.md#mariadb-jdbc-framework) as a runtime dependency.

However, we're going to explicitly define a runtime dependency on the MySQL JDBC driver.

<details>
<summary>*Gradle Help*</summary>

Open **build.gradle** for editing and add the following to the *dependencies* section

```
runtime('mysql:mysql-connector-java:8.0.14')
```
</details>

<details>
<summary>*Maven Help*</summary>

Open **pom.xml** for editing and add the following to the *dependencies* section

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.14</version>
    <scope>runtime</scope>
</dependency>
```
</details>

* And, of course we must rebuild and repackage the application to have the application recognize the new dependency at runtime

```bash
mvn package
```

* Let's bind the service to the application, e.g.

```bash
cf bind-service cloud-native-spring mysql-database
```

* Expected output:

```bash
Binding service mysql-database to app cloud-native-spring in org zoo-labs / space development as cphillipson@pivotal.io...
OK
```

-> Tip: Use `cf restage cloud-native-spring` to ensure your env variable changes take effect

<br>

* Now let's push the updated application

```bash
cf push cloud-native-spring
```

* You may wish to observe the logs and notice that the bound MySQL database is picked up by the application, e.g.

```bash
cf logs cloud-native-spring --recent
```

* Sample output:

```bash
...
INFO 20 --- [           main] org.hibernate.Version                    : HHH000412: Hibernate Core {5.0.12.Final}
INFO 20 --- [           main] org.hibernate.cfg.Environment            : HHH000206: hibernate.properties not found
INFO 20 --- [           main] org.hibernate.cfg.Environment            : HHH000021: Bytecode provider name : javassist
INFO 20 --- [           main] o.hibernate.annotations.common.Version   : HCANN000001: Hibernate Commons Annotations {5.0.1.Final}
INFO 20 --- [           main] org.hibernate.dialect.Dialect            : HHH000400: Using dialect: org.hibernate.dialect.MySQLDialect
INFO 20 --- [           main] org.hibernate.tool.hbm2ddl.SchemaUpdate  : HHH000228: Running hbm2ddl schema update
...
```

* You could also bind to the database directly from the `manifest.yml` file, e.g.

```yaml
applications:
- name: cloud-native-spring
  random-route: true
  memory: 1024M
  instances: 1
  path: ./build/libs/cloud-native-spring-1.0-SNAPSHOT-exec.jar
  buildpacks:
  - java_buildpack_offline
  timeout: 180 # to give time for the data to import
  env:
    JAVA_OPTS: -Djava.security.egd=file:///dev/urandom
  services:
    - mysql-database
```

* Attempt to push the app again after making this update

```bash
cf push
```

* Let's have a look at how we can interact with the database

Visit [Pivotal MySQL*Web](https://github.com/pivotal-cf/PivotalMySQLWeb) then follow these instructions for building the application.

```bash
cd ..
git clone https://github.com/pivotal-cf/PivotalMySQLWeb.git
cd PivotalMySQLWeb
./mvnw -DskipTests=true package
```

* Then to prepare the application for deployment we'll create a manifest. Open an editor, create and save a file named `manifest.yml` with these contents:

```yaml
applications:
- name: pivotal-mysqlweb
  memory: 1024M
  instances: 1
  random-route: true
  path: ./target/PivotalMySQLWeb-1.0.0-SNAPSHOT.jar
  services:
    - mysql-database
  env:
    JAVA_OPTS: -Djava.security.egd=file:///dev/urandom
```

* Of course, you'll want to deploy the application

```bash
cf push
```

* And once deployed, you can visit the appliation URL and log in with the default credentials `admin/cfmysqlweb`

Take a few moments to explore the features and see that the administrative and diagnostic functions of Pivotal MySQL*Web provide a rather simple way to interact with and keep your database instance up-to-date via an Internet browser.
