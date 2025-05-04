# Spring-Boot-Audit-Demo


## 1. What is Auditing?

Auditing, in the context of software development, means tracking changes made to data. 
This is crucial for many real-world applications. 
Think of it as a detailed history of your data. 

Key questions that auditing helps answer:

* Who created a record?
* When was it created?
* Who updated it?
* When was it updated?
* What specific changes were made?

This information is invaluable for:

* **Security:** Understanding who accessed or modified sensitive data.
* **Debugging:** Pinpointing the cause of data corruption or unexpected behavior.
* **Compliance:** Meeting regulatory requirements that mandate data change tracking.
* **Business Intelligence:** Analyzing how data evolves over time.

## 2. What is Hibernate Envers?

Hibernate Envers is a powerful tool that integrates with Hibernate (the most popular JPA implementation) to automate the process of auditing.  

Instead of you having to manually create audit tables and write code to track changes, Envers does it for you.

Here's how it works:

* **Automatic History Tables:** Envers automatically creates shadow tables (often named with a `_AUD` suffix) to store historical versions of your data.
* **Change Tracking:** When you create, update, or delete an entity, Envers intercepts these actions and creates a snapshot of the data *before* the change, saving it in the audit table.
* **Data Retrieval:** Envers provides an API that allows you to easily query these audit tables and retrieve historical versions of your data.  You can see exactly how a record looked at any point in its history.

## 3. What We Will Do in This Demo

This demo will show you how to set up auditing in a Spring Boot application using Hibernate Envers.  We'll cover these key aspects:

* **Spring Boot + JPA:** We'll use Spring Boot's Data JPA module to interact with our database.
* **Enable Auditing:** We'll configure Spring Boot to activate auditing.
* **Track Created/Updated Fields:** We'll automatically track who created and updated records, and when.
* **Track Data History with Envers:** We'll use Envers to create `_AUD` tables and store a complete history of changes.

## 4. Project Setup (Dependencies)

To get started, we need to include the necessary dependencies in our project's build configuration.  For a Maven project, this is done in the `pom.xml` file.  We need:

* **Spring Data JPA Starter:** This provides the foundation for using Spring Data JPA with Hibernate.
* **Hibernate Envers:** This is the core Envers library that provides the auditing functionality.
* **H2 Database:** For this demo, we'll use H2, an in-memory database, which is convenient for quick setup.  In a real-world application, you would replace this with a production database like MySQL or PostgreSQL.

## 5. Enable Auditing in the Application

To tell Spring Boot that we want to use auditing, we create a configuration class and use the `@EnableJpaAuditing` annotation.  
This annotation tells Spring to enable JPA auditing and also lets us specify who will be doing the auditing using `auditorAwareRef`.

## 6. Create AuditorAware Bean

The `AuditorAware` interface is a crucial part of Spring Data JPA auditing. 
It's responsible for providing the *current user* information whenever an audited entity is created or modified. 
For example, it might return the username of the logged-in user.

We need to create a class that implements `AuditorAware` and then register it as a Spring bean.  This bean will then be used by Spring whenever it needs to know the current auditor.

In a real-world application, you would typically retrieve the current user's information from Spring Security or your application's authentication context.  For simplicity, in this demo, we'll use a hardcoded value.

## 7. Create Base Auditable Entity

To avoid having to add the same auditing fields (`createdBy`, `createdDate`, `updatedBy`, `updatedDate`) to every entity in our application, we create a base class called `Auditable`.

This class contains these common fields, annotated with Spring Data JPA's auditing annotations:

* `@CreatedBy`:  Automatically populated with the user who created the entity.
* `@CreatedDate`:  Automatically populated with the date and time when the entity was created.
* `@LastModifiedBy`:  Automatically populated with the user who last modified the entity.
* `@LastModifiedDate`:  Automatically populated with the date and time when the entity was last modified.

By having our entities extend this `Auditable` class, they automatically inherit these fields and the auditing behavior.

## 8. Create Entity with Envers Audit

To enable Envers auditing for a specific entity, we annotate that entity class with the `@Audited` annotation.

For example, if we have an `Employee` entity, annotating it with `@Audited` will tell Envers to:

* Create an audit table (e.g., `EMPLOYEE_AUD`) to store historical versions of `Employee` data.
* Automatically create a snapshot of the Employee data in the `EMPLOYEE_AUD` table whenever an Employee record is inserted, updated, or deleted.

## 9. H2 Database Setup - or any database of your choice

For this demo, we'll use an H2 in-memory database.  We configure the connection details in the `application.properties` file.  This file also includes a setting to tell Envers to store data even when a record is deleted.

## 10. How Audit Tables Work

Envers works by creating audit tables that mirror your audited entity tables.  For each audited entity (e.g., `Employee`), Envers creates a corresponding audit table (e.g., `EMPLOYEE_AUD`).

When you perform a database operation (insert, update, delete) on an audited entity, Envers does the following:

* **Insert/Update:** Envers inserts a new record into the audit table, containing the data *before* the change.  For updates, it's the data before the update.
* **Delete:** When you delete a record, Envers, depending on the configuration, either inserts a final record into the audit table with the data before deletion or marks the existing audit record as deleted.
* **Revision Information:** Each record in the audit table includes a `REV` column (revision ID), which is a foreign key to a `REVINFO` table.  This `REVINFO` table stores metadata about the revision, such as the timestamp.
* **Revision Type:** The audit table also includes a `REVTYPE` column, which indicates the type of operation that caused the revision:
    * 0: INSERT
    * 1: UPDATE
    * 2: DELETE

## 11. How REVINFO Table Looks

The `REVINFO` table is automatically managed by Envers.  It stores information about each revision (i.e., each change to an audited entity).  It typically contains:

* `REV` (Revision ID):  A unique identifier for the revision.  This is the primary key of the `REVINFO` table and is referenced by the audit tables.
* `REVTSTMP` (Revision Timestamp):  The date and time when the revision occurred.

Every time a change is made to an audited entity, a new entry is added to the `REVINFO` table, and the corresponding audit table record is linked to that entry via the `REV` column.

## 12. How You Can Fetch Old Data

Envers provides an API (through the `AuditReader` interface) that allows you to query the audit tables and retrieve historical versions of your data.

For example, you can write a query to get all the revisions of a specific employee, showing how their data has changed over time.

## 13. `@Audited` Related Annotations

* **`@Audited`**: This is the core annotation.  When placed on an entity class, it tells Hibernate Envers to track all changes to that entity's data.  Envers will create a corresponding audit table (typically named `[EntityName]_AUD`) to store the history.

* **`@NotAudited`**:  You can use this annotation on a specific field within an `@Audited` entity to exclude that field from being tracked by Envers.  This is useful for fields that you don't need to audit (e.g., a large binary field or a field that changes very frequently and you don't care about its history).

* **`@AuditOverride`, `@AuditOverrides`**: These annotations are used when your entity inherits fields from a superclass (e.g., the `Auditable` base class we created).  They allow you to override the auditing behavior of the inherited fields.
    * `@AuditOverride`:  Used to override the auditing behavior of a *single* inherited field.  For example, you could use `@AuditOverride(name = "createdBy", isAudited = false)` on an entity to prevent the `createdBy` field from being audited for that specific entity.
    * `@AuditOverrides`:  Used to apply multiple `@AuditOverride` annotations at once, if you need to override the auditing behavior of several inherited fields.
