+++
title = "Leveling Up with SQL: Using Dapper, Result Sets, and Temp Tables"
date = 2022-10-24
[taxonomies]
authors = ["Elias Prescott"]
tags = ["SQL", "C#", "Dapper"]
+++

Persisting data in a database is an extremely common requirement in many types of applications. For the majority of use cases, querying databases should be a simple task. Most database queries should be easy to read and write and quick to run, but there are some exceptions.

A notable exception that I ran into recently comes when I needed to query large amounts of nested objects. Querying a list of objects is fairly simple, but it becomes more complicated when each object in the list also has a list of child objects stored in a different DB table. Then, if those child objects can have their own nested child objects stored in a third table, things become even more complicated.

It is definitely possible to solve this kind of challenge without any SQL wizardry, but there are some big advantages to digging deeper and using some advanced techniques. So, I want to go over a basic example and show some advanced SQL techniques that you can use to level up your queries.

For this example, I will create a simple C# console application. I will also be using [Dapper](https://www.learndapper.com/) to simplify our DB connection code. You will probably need to manually add ```System.Data.SqlClient``` to use the ```SqlConnection``` class. I'll start by initializing the project and adding the starting dependencies:

```bash
dotnet new console -o mighty-query-example

cd mighty-query-example

dotnet add package System.Data.SqlClient --version 4.8.3

dotnet add package Dapper --version 2.0.123
```

For our example scenario, let's say that we are modeling lists of vehicles, their passengers, and any items those passengers are holding. To make things more complicated, our vehicle class will also be able to hold its own list of items as well. To make things even more complicated, each vehicle will be able to tow (or contain) another vehicle. Here is the final model I came up with:

```c#
// Models.cs

public class Vehicle {
    public int Id { get; set; }
    public string? Make { get; set; }
    public string? Model { get; set; }
    public int? Year { get; set; }
    public int? TowingVehicleId { get; set; }
    public List<Passenger> Passengers { get; set; } = new List<Passenger>();
    public List<Item> Items { get; set; } = new List<Item>();
    public Vehicle? TowedVehicle { get; set; }
}

public class Passenger {
    public int Id { get; set; }
    public string? FirstName { get; set; }
    public string? LastName { get; set; }
    public int? VehicleId { get; set; }
    public List<Item> Items { get; set; } = new List<Item>();
}

public class Item {
    public int Id { get; set; }
    public string? Title { get; set; }
    public string? Description { get; set; }
    public int? PassengerId { get; set;}
    public int? VehicleId { get; set;}
}
```

After setting up a table for each of these classes, we need to populate the tables with some dummy data. For this, I will be using [Faker.Net](https://github.com/Kuree/Faker.Net).

```bash
dotnet add package Faker.Net --version 2.0.154
```

We can use Faker to generate random string data and we can use ```System.Random``` for generating IDs and other numerical data. So, we have everything ready to generate data, but we will need a method for inserting our mock data into the database. One possible method would be to build a bulk SQL insert statement ourselves. This approach would work and would be perfectly suitable for inserting smaller groups of data, but we will be inserting 10,000 or more rows into each of our tables. Because we are inserting so much data at once, I would be more comfortable with using a standardized solution. We are in luck, because ```System.Data.SqlClient``` provides a class for exactly this kind of problem. The ```SqlBulkCopy``` class will allow us to bulk insert our rows into a connected database as efficiently as possible. So, we have identified our solutions for generating data and inserting it, now we just need to put them in practice.

```c#
// DummyData.cs

using System.Data;
using System.Data.SqlClient;

public static class TablePopulation {

    // For ease of use, I am making this an extension method on SqlClient.
    public static void PopulateTable(
        this SqlConnection connection,
        string tableName,
        int numberOfRows,
        // The Func<string?> value is our data generator method.
        params (string, Func<string?>)[] columnList
    ) {
        // Create a local blank table to fill with data.
        DataTable table = new DataTable();

        // Initialize the bulk copy handler.
        SqlBulkCopy bulkCopy = new SqlBulkCopy(
            connection,
            SqlBulkCopyOptions.TableLock |
            SqlBulkCopyOptions.FireTriggers |
            SqlBulkCopyOptions.UseInternalTransaction,
            null
        );

        // Set the target table name.
        bulkCopy.DestinationTableName = tableName;

        foreach (var (columnName, _generator) in columnList) {
            // Add the column definition to our local table.
            table.Columns.Add(columnName);

            // Add a column mapping to our copier.
            // If you don't add these mappings, the columns will be matched by index.
            bulkCopy.ColumnMappings.Add(columnName, columnName);
        }

        // Add the rows to the local table
        for (int i = 0; i < numberOfRows; i++) {
            object[] rowValues = columnList
                // Here is where we use the data generator functions.
                // If the generator is null, then we just pass on the null.
                .Select(col => col.Item2 is null ? null : col.Item2())
                .Cast<object>()
                .ToArray();

            // You could try to build a DataRow manually and insert it here,
            // but Rows.Add() is perfectly happy accepting an object array instead.
            table.Rows.Add(rowValues);
        }

        connection.Open();

        bulkCopy.WriteToServer(table);

        connection.Close();
    }
}
```

Going back to our model, I marked some of the foreign ID properties as nullable because I wanted those relations to be optional. This means that when we generate integers for those IDs, we will want some of the integers to be null. To make this easier, I wrote a quick extension method for ```System.Random```

```c#
// Utilities.cs

public static class Utilities {
    public static bool NextBool(this Random rnd, int truePercentChance = 50) =>
        rnd.Next(100) <= truePercentChance;
}
```

We now have everything in place to finally generate some mock data.

```c#
// Program.cs

using System.Data.SqlClient;

var rnd = new Random();

using (var connection = new SqlConnection(/* Connection string here */)) {
    connection.PopulateTable(
        "Vehicles",
        10_000,
        ("Make", Faker.Name.Middle),
        ("Model", Faker.Name.Middle),
        ("Year", () => rnd.Next(1776, 3000).ToString()),
        (
            "TowingVehicleId", 
            () => rnd.NextBool() ? rnd.Next(1, 10_000).ToString() : null
        )
    );

    connection.PopulateTable(
        "Passengers",
        20_000,
        ("FirstName", Faker.Name.First),
        ("LastName", Faker.Name.Last),
        (
            "VehicleId", 
            () => rnd.NextBool(90) ? rnd.Next(1, 10_000).ToString() : null
        )
    );

    connection.PopulateTable(
        "Items",
        50_000,
        ("Title", Faker.Company.Name),
        ("Description", Faker.Company.CatchPhrase),
        (
            "PassengerId",
            () => rnd.NextBool() ? rnd.Next(1, 20_000).ToString() : null
        ),
        (
            "VehicleId",
            () => rnd.NextBool(10) ? rnd.Next(1, 10_000).ToString() : null
        )
    );
}
```

Using the ```params``` keyword on ```PopulateTable()``` allows us to pass in any number of table definitions when we call it. The usage of ```Func<string?>``` for the generators also allows for maximum flexibility. You can see how we are able to pass in the Faker methods directly and how we can define our own lambdas for generating random numbers.

We are generating and inserting 80,000 rows in this code snippet, so it will take a couple of seconds to run at a minimum. I haven't tried to benchmark or optimize any of this code, but I am guessing our generator functions will be the biggest slow-down. If you are wanting to insert millions of rows or more, I would recommend looking into optimizing the data generation, or just inserting static data instead.

We are well into this tutorial about writing SQL queries, but we still haven't written any ourselves. Let's fix that and start querying our tables.

```c#
// Program.cs

using System.Data.SqlClient;
using Dapper;

using (var connection = new SqlConnection(/* Connection string here */)) {
    List<Vehicle> vehicles = connection
        .Query<Vehicle>("SELECT * FROM Vehicles")
        .AsList();

    Console.WriteLine(vehicles.Count);
    // Result should be: 10000
}
```

If you aren't familiar with using Dapper, it just adds ```Query()``` and various other extension methods designed to simplify database connection code. We just tell Dapper what we are expecting to query and it will do all the hard work of retrieving and mapping the rows.

Getting a list of vehicles is nice, but what if we want more information? Next, let's try to get a list of vehicles, along with a list of passengers for each vehicle. To do this, our query will need to return multiple result sets.

```c#
string sql = @"
    SELECT * FROM Vehicles

    SELECT * FROM Passengers 
    WHERE Passengers.VehicleId IN (SELECT Id FROM Vehicles)
";

// Load multiple result sets 
var data = connection.QueryMultiple(sql);

// Read the first result set of vehicles
List<Vehicle> vehicles = data.Read<Vehicle>().AsList();

// Read the second result set of passengers
List<Passenger> passengers = data.Read<Passenger>().AsList();

// Find the list of passengers that are in each vehicle
foreach (var vehicle in vehicles) {
    vehicle.Passengers = passengers
        .Where(p => p.VehicleId == vehicle.Id)
        .ToList();
}
```

To read multiple result sets, we can use ```QueryMultiple()``` to load all results into one object. From there, you just need to read the result sets out of the data object in the right order. After we read our list of vehicles and passengers, we can iterate through the vehicles and assign each vehicle its list of passengers. Understanding how to use results sets is fantastic because it allows you to combine multiple related queries into one. Reducing the number of database calls is good for reducing DB load, but it can also speed things up on the application side.

Now, let's say that we want to find all passengers with a specific first name. For that, we can use some SQL like this:

```sql
SELECT * FROM Passengers 
WHERE FirstName = 'Glenda'
```

Now what if we want more information about these passengers named Glenda? What if we want a list of vehicles they are in?

```sql
SELECT * FROM Passengers 
WHERE FirstName = 'Glenda'

SELECT * FROM Vehicles
WHERE Id in (SELECT VehicleId FROM Passengers 
	WHERE FirstName = 'Glenda')
```

This works and seems fairly performant, but it doesn't look the greatest. Duplicating code is generally a no-no and I can't help but feel that SQL queries should be no exception to that rule. This query will start looking even worse when we need to get a list of items that all our Glenda's are holding.

```sql
SELECT * FROM Passengers 
WHERE FirstName = 'Glenda'

SELECT * FROM Vehicles
WHERE Id in (SELECT VehicleId FROM Passengers 
	WHERE FirstName = 'Glenda')

SELECT * FROM Items
WHERE PassengerId IN (SELECT Id FROM Passengers 
	WHERE FirstName = 'Glenda')
```

Again, this SQL does the job and is not difficult to write, but it can be difficult to read it due to all the nested queries. I'll make it even worse by also querying any items that are in the vehicles that have a passenger named Glenda.

```sql
SELECT * FROM Passengers 
WHERE FirstName = 'Glenda'

SELECT * FROM Vehicles
WHERE Id in (SELECT VehicleId FROM Passengers 
	WHERE FirstName = 'Glenda')

SELECT * FROM Items
WHERE PassengerId IN (SELECT Id FROM Passengers 
	WHERE FirstName = 'Glenda')
	OR
	VehicleId IN (SELECT Id FROM Vehicles 
		WHERE Id IN (SELECT VehicleId FROM Passengers
			WHERE FirstName = 'Glenda'))
```

It is possible to read and comprehend this query, but it's starting to lose maintainability in my opinion. Imagine maintaining a legacy app that was filled with queries like this. I certainly would not enjoy maintaining this query, and I don't think many other developers would either. This kind of query makes me sad because SQL is a very cool (and very legal) language. It shouldn't have to be like this.

Let's make me happy again by fixing this query. We'll make it more readable by using my new favorite SQL feature: temp tables.

Temp tables are exactly what they sound like. They are just temporary tables. They allow us to stash the results of a query into a temporary variable for use later on. As far as I know, temp tables are saved per connection, which means you don't have to worry about temp table name collisions. It is possible to create a temp table in one query and access it from another query, but we will not be doing that here. Instead, we will drop all our temp tables at the end of our query. We are not trying to persist temp data long-term, we only want to avoid repetition in our sub-queries.

```sql
SELECT * INTO #tempPassengers
FROM Passengers 
WHERE FirstName = 'Glenda'

SELECT * INTO #tempVehicles
FROM Vehicles
WHERE Id in (SELECT VehicleId FROM #tempPassengers)

SELECT * FROM #tempPassengers

SELECT * FROM #tempVehicles

SELECT * FROM Items
WHERE 
    PassengerId IN (SELECT Id FROM #tempPassengers)
    OR
    VehicleId IN (SELECT Id FROM #tempVehicles)

DROP TABLE #tempPassengers
DROP TABLE #tempVehicles
```

This new query is not perfect, but I think it is easier to read and maintain than the previous version. Notice that we still need to query the temp tables to return their results. Selecting into a temp table will not return it by default. You have to use a separate ```SELECT``` to return all columns for that temp table.

I hope you enjoyed reading this post. To recap, we learned how to mock and insert bulk amounts of data, how to use Dapper, and how to use SQL result sets and temp tables. Please, [make an issue](https://github.com/EliasPrescott/eliasprescott.github.io/issues/new) if you think I missed anything. I learned all these SQL features very recently, so I am still learning how to use them myself. I think a good understanding of result sets and temp tables is essential for any developer that works with SQL. I have only been using them for a few weeks, but these features have already made my job much easier.