# Migration

- [Introduction](#introduction)
- [Create a new migration](#create-migration)
- [Run migration](#run-migration)


<a name="introduction"></a>
## Introduction
The Going migration is build to be very simple, migration. that support migration for SqLite and Postgress. 


<a name="#create-migration"></a>
## Create migration
Creating migration in GOING you can use the build in CLI tool art. by running the command ``` art make:migration post``` This will create a new migration file, and place it in the folder ``` internal/database/migrations/scripts```
The file will be a simple SQL file, where you can add setup for your DB. 



<a name="#run-migration"></a>
## Run migration
For running migration, run ```art migrate```and then art will take the DB connectionn you defined in your .env file, and make the migration to DB.  

