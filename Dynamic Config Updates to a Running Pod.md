## Dynamic Config Updates to a Running Pod

In this chapter, we will learn to update configuration in a running pod without redeploying the pod.  As an example we'll the database connection to use a PostgreSQL database.

**Use Case:** As a developer, I need to change quick configuration changes while building my application. I don't want to redeploy the application pod each time.

In the previous lab, we created a configuration to connect to the MySQL database by mounting `application.properties` file using ConfigMap. Here we will add a PostgreSQL database and change the configuration dynamically.

**Step 1: Add PostgreSQL Database Pod**

Use Ensure you are in the `myproject` project. 

Based on what you have learnt in the previous labs, create a new Postgresql database pod i.e. `Add to project`, search for `postgresql`, choose `postgresql-ephemeral`. You will need to key in the following values.

**Database Service Name:** postgresql		
**MySQL Connection Username:** user		
**MySQL Connection Password:** password		
**MySQL Database Name:** sampledb	

You should see the following message on the screen once the database is created.

```
The following service(s) have been created in your project: postgresql.

       Username: user
       Password: password
  Database Name: sampledb
 Connection URL: mysql://postgresql:5432/
```


**Step 2: Add data to the database**

While the application code has `schema-postgresql.sql` and `data-postgresql.sql` files, these scripts will only be executed if the pod restarts. Since we are going to use this database without restarting the bootapp pod, let's add the data manually.

Select the `postgresql` pod and navigate to the `terminal`.

Login to the database from CLI

```
psql -h 127.0.0.1 -U $POSTGRESQL_USER -q -d $POSTGRESQL_DATABASE
```
This will log you into the psql CLI.

Now create the table using the script
```
CREATE TABLE IF NOT EXISTS customer (
   CUST_ID serial primary key,   
   NAME varchar(100) NOT NULL,
   AGE integer NOT NULL);
```

and then add some data as shown below. You can change the values if you wish.

```
insert into customer (name,age) values ('Joe Psql', 88);
insert into customer (name,age) values ('Jack Psql', 54);
insert into customer (name,age) values ('Ann Psql', 32);
```

These scripts are available here -
[https://github.com/RedHatWorkshops/spring-sample-app/blob/master/src/main/resources/schema-postgresql.sql](https://github.com/RedHatWorkshops/spring-sample-app/blob/master/src/main/resources/schema-postgresql.sql)

[https://github.com/RedHatWorkshops/spring-sample-app/blob/master/src/main/resources/data-postgresql.sql](https://github.com/RedHatWorkshops/spring-sample-app/blob/master/src/main/resources/data-postgresql.sql)

Here is how I added the data in the terminal


```
sh-4.2$ psql -h 127.0.0.1 -U $POSTGRESQL_USER -q -d $POSTGRESQL_DATABASE                                                                                               
sampledb=> CREATE TABLE IF NOT EXISTS customer (                                                                                                                       
sampledb(>    CUST_ID serial primary key,                                                                                                                              
sampledb(>    NAME varchar(100) NOT NULL,                                                                                                                              
sampledb(>    AGE integer NOT NULL);                                                                                                                                   
sampledb=> insert into customer (name,age) values ('Joe Psql', 88);                                                                                                    
sampledb=> insert into customer (name,age) values ('Jack Psql', 54);                                                                                                   
sampledb=> insert into customer (name,age) values ('Ann Psql', 32);                                                                                                    
sampledb=> select * from customer;                                                                                                                                     
 cust_id |   name    | age                                                                                                                                             
---------+-----------+-----                                                                                                                                            
       1 | Joe Psql  |  88                                                                                                                                             
       2 | Jack Psql |  54                                                                                                                                             
       3 | Ann Psql  |  32                                                                                                                                             
(3 rows)                                                                                                                                                               
                                                                                                                                                                       
sampledb=> \q                                                                                                                                                        
                                                                                                                                                                                                                                                                                                                                                                         
```

**Step 3: Update ConfigMap**

Now let us update the ConfigMap to change the `application.properties` to point to `postgresql` datasource.

Using CLI run the following command in the `myproject`.

```
$ oc edit configmap app-props
```
Or via Web Console navigate to
- Menu item `Resources`->`Other Resources`
- Choose `ConfigMap`
Look at the `app-props` in the table towards the right and click on `Action`->`Edit YAML`

Edit the datasource parameters as below

```
spring.datasource.url=jdbc:jdbc:postgresql://postgresql:5432/sampledb                                                                                                 
spring.datasource.platform=postgresql                                                                                                                                                                                                                            
spring.datasource.username=user                                                                                                                                        
spring.datasource.password=password
```
**Do this slowly. Double check every parameter**

Specifically note the `spring.datasource.url`. It is in the following format:

`
spring.datasource.url = jdbc:<<databasetype>>://<<service-host>>:<<service-port>>/<<dbname>>
`

Here 			

* databasetype is `postgresql`			
* service-host can be ip-address of the service or the service name. 		
* service-port for postgresql is `5432`

**Note** that this change does not redeploy the pod. You can check the pod details to see how long the pod has been running for.

Now verify the `application.properties` inside the pod (Go to bootapp pod terminal on the Web Console or use oc rsh) . You will note that the `application.properties` file is now updated as a result of updating the ConfigMap.

```                                                                               
sh-4.2$ cat config/application.properties                                                                                                                                                                                                                                                                                                                          
spring.datasource.url=jdbc:jdbc:postgresql://postgresql:5432/sampledb                                                                                                 
spring.datasource.platform=postgresql                                                                                                                                                                                                                            
spring.datasource.username=user                                                                                                                                        
spring.datasource.password=password
```


**Step 4: Test the application connection**

Click the application url now i.e `http://bootapp-spring-UserName.apps.devday.ocpcloud.com/`. It will open a new tab and greets you

```
Hello from bootapp-2-06a4b
```

Also watch the pod logs either using web console or using CLI. For example `oc logs -f bootapp-2-06a4b`
Watch out for connection url in the output.

Now try the `/dbtest` endpoint i.e. `http://bootapp-myproject.192.168.0.102.xip.io/dbtest`.

**Note** that the output is still from the MySQLDB.

```
Customers List


CustomerId: 2 Customer Name: Joe Mysql Age: 88
CustomerId: 3 Customer Name: Jack Mysql Age: 54
CustomerId: 4 Customer Name: Ann Mysql Age: 32
```

Also the pod logs show that connection url is

```
connection url: jdbc:mysql://mysql:3306/sampledb?useSSL=false
```
So even after the `application.properties` file is updated in the pod, the new values are not picked up. The reason is that springboot app caches the environment variables. This application has a `@RefreshScope` annotation. So we can invoke `/refresh` endpoint to refresh the cache. Run the following command from CLI to refresh the cache.


```
$ curl -X POST http://bootapp-myproject.192.168.0.102.xip.io/refresh
["spring.datasource.url","spring.datasource.platform"]
```

Note that the pod logs show that the refresh is executed.

```
2017-02-10 15:18:52.109  INFO 9 --- [nio-8080-exec-7] s.c.a.AnnotationConfigApplicationContext : Refreshing 
```

Now try the `/dbtest` endpoint again. Now the result will show the data from the postgresql database.

```
Customers List


CustomerId: 1 Customer Name: Joe Psql Age: 88
CustomerId: 2 Customer Name: Jack Psql Age: 54
CustomerId: 3 Customer Name: Ann Psql Age: 32
```

```

Also note the logs will show the connection url as
```
connection url: jdbc:postgresql://postgresql:5432/sampledb
```
**Note** in this exercise, the pod was never redeployed. The application.properties were dynamically updated.


**Conclusion:** In this chapter, we have learnt the ConfigMap's flexibility and how it allows dynamic updates to the pod configuration.
