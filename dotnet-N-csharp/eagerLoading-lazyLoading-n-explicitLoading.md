# Eager Loading , Explicit Loading and Lazy Loading

These three are the strategies, how we retrieve related data from the database using ORM like Entity Framework Core. 

## 1. Eager Loading

Load the main entity and its related data when query is executed. ORM include related data using join in SQL and it fatch all data at once. In Entity Framework Core
.Inclide() is used to load the related entities.

## 2. Explicit Loading
Load only main entity when query is executed, for loading related data needs to use seperate commands. In Entity Framework Core Load() is used for quering related data explicitly. 


## 3. Lazy Loading

