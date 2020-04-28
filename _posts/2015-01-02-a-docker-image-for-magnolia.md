---
layout: post
title: "Deploying Magnolia on CloudFoundry"
date: 2015-01-02 00:00:00 -0000
---
Cloud Foundry is platform as a service (PaaS) and provides a technical stack that includes the run-time and services on top of a cloud infrastructure. 

Cloud Foundry imposes certain constraints on applications in order to maximize its scalability and reliability. For instance, applications have to be stateless and data have to be stored and managed by services.

At Magnolia, we developed a [dedicated bundle](https://documentation.magnolia-cms.com/display/DOCS/Cloud+Foundry+Integration) to ease the deployment of our CMS to Cloud Foundry.

##Installation
To deploy an existing project to Cloud Foundry, your pom files must simply be updated to use the cloudfoundry bundle instead of the default bundle.

Here is a quick tutorial if you start from a blank page.

First, create a Magnolia project as usual with our [maven archetype](http://wiki.magnolia-cms.com/display/WIKI/Module+QuickStart): 

```prettyprint lang-bsh
mvn archetype:generate -DarchetypeCatalog=https://nexus.magnolia-cms.com/content/groups/public/
```
Update the dependencies in the parent pom file in order to use the cloudfoundry bundle instead of the default bundle:
```prettyprint lang-xml
<dependencyManagement>
    <dependency>
        <groupId>info.magnolia.cloudfoundry</groupId>
        <artifactId>cloudfoundry-bundle-ce-webapp</artifactId>
        <version>${cloudfoundryVersion}</version>
        <type>war</type>
    </dependency>
    <dependency>
        <groupId>info.magnolia.cloudfoundry</groupId>
        <artifactId>cloudfoundry-bundle-ce-webapp</artifactId>
        <version>${cloudfoundryVersion}</version>
        <type>pom</type>
    </dependency>
    <dependency>
        <groupId>info.magnolia.cloudfoundry</groupId>
        <artifactId>cloudfoundry-bundle-ce</artifactId>
        <version>${cloudfoundryVersion}</version>
        <type>pom</type>
        <scope>import</scope>
    </dependency>
</dependencyManagement>
```
Replace the same depenencies in the Magnolia webapp pom file:
```prettyprint lang-xml
<dependencies>
    <dependency>
        <groupId>info.magnolia.cloudfoundry</groupId>
        <artifactId>cloudfoundry-bundle-ce-webapp</artifactId>
        <type>war</type>
    </dependency>
    <dependency>
        <groupId>info.magnolia.cloudfoundry</groupId>
        <artifactId>cloudfoundry-bundle-ce-webapp</artifactId>
        <type>pom</type>
    </dependency>
</dependencies>
```
Build the project, the resulting war file can be deployed on Cloud Foundry:
```prettyprint lang-bsh
cf push my-magnolia-project -p target/my-magnolia-project.war
```

##How it works?
The configuration of the webapp is based on environment variables and leverages our single war deployment approach.

The environment variable ```magnoliaInstanceType``` must be set in Cloud Foundry to point to one of the available configurations defined in the config folder of the webapp - For instance ```magnoliaAuthor``` or ```magnoliaPublic``` will create one author and public instances using derby as a database.

In order to use a separate service to persist the content - such as a MySQL or postgreSQL databases - the ```VCAP_SERVICES``` environment variable is automatically injected in Magnolia as multiple system properties:

- The username of the third instance of the MySQL service : ```vcap.services.mysqldb.db3.credentials.username```
- The URI of the first instance of the ElephantSQL service ```vcap.services.elephantsql.esql1.credentials.uri```

These system properties can be used in the repository configuration to point dynamically to the right service instance:

```prettyprint lang-xml
 <DataSource name="magnolia">
 	<param name="driver" value="com.mysql.jdbc.Driver" />
	<param name="url" value="${vcap.services.mysqldb.db1.credentials.uri}" />
	<param name="user" value="${vcap.services.mysqldb.db1.credentials.username}" />
	<param name="password" value="${vcap.services.mysqldb.db1.credentials.password}" />
	<param name="databaseType" value="mysql"/>
	<param name="validationQuery" value="select 1"/>
 </DataSource>
```
