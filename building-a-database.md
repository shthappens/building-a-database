### Introduction

You are tasked with taking old CSV data and placing that info into a nice, normalized database.
The CSV data represents buildings and their owners. Each record has a name, a zoning type, and a construction type.
Notice the duplication of data in the latter two columns.  

#### Resources

- [Ruby's CSV library](http://ruby-doc.org/stdlib-2.3.0/libdoc/csv/rdoc/CSV.html)
- [TutorialsPoint Postgres Reference](https://www.tutorialspoint.com/postgresql/index.htm)

#### Learning Goals

- What does database normalization look like? We will compare and contrast CSV data storage and relational database storage.
- How do we construct `INSERT` and `SELECT` queries in a meaningful way? Does the order in which we insert records matter?
- What does the term *'dependency'* mean in the context of a database?

### Instructions

#### Setup

```no-highlight
$ et get building-a-database
```

This exercise requires the `pg` gem. If you don't already have it installed, let's do that:

```no-highlight
$ gem install pg
```

This exercise will require the creation of a database:

```no-highlight
$ createdb building_database
```

#### Create the Database Schema

For this exercise, you will want create three tables: `accounts`, `zoning_types`, and `construction_types`.
Use the provided `schema.sql` to store your SQL code for creating the **schema** for this database.

```sql
CREATE TABLE zoning_types (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255)
);
```

Try to create SQL code that will generate the other two tables on your own.
Don't import the schema yet! There are other steps to take before that

{% show_hint %}

Take a look at the rows and columns in the `data.csv` file. This information
should help you to determine what the database table column names should be, as
well as the datatypes stored in those columns should be.

Ask yourself the question, "How will these three tables relate to each other?"
We need a way _relate_ records in the `accounts` table with records in the
`construction_types` and `zoning_types` tables. Which table will store the
**foreign keys** to create the necessary relationships?

{% endshow_hint %}

{% show_solution %}
```sql
CREATE TABLE construction_types (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255)
);

CREATE TABLE accounts (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255),
  zoning_type_id INTEGER,
  construction_type_id INTEGER
);
```

If this doesn't make sense, take a look at the chart further down in this lesson to get an idea of what tables and columns this code is building and how they work together.
{% endshow_solution %}

#### Drop It Like It's Hot

Once you are ready to apply this schema to your newly created database, import the schema like so:

```no-highlight
$ psql building_database < schema.sql
```

If you are trying this step on your own without the solution, importing the schema may take you more than one attempt, and you may realize that you need to alter your schema as you go.
If you run the schema command more than once, you will probably get errors like this:

```no-highlight
Relation 'zoning_types' already exists.
```

This error is a safety feature.
This is Postgres telling you, "Hey, if you're going to overwrite this table and the data it contains, you should be explicit about it."
The issue is that we are not being explicit enough with our code.

To ensure that the schema file imports correctly *each time*, **you will want to provide explicit instructions to drop all of your tables at the beginning of your schema file**:

```sql
DROP TABLE IF EXISTS accounts;
```

You will want one sql query like this for each table. When you run the import command again, you should get this output:

```no-highlight
DROP TABLE
DROP TABLE
DROP TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
```

{% show_solution %}

```sql
DROP TABLE IF EXISTS accounts;
DROP TABLE IF EXISTS zoning_types;
DROP TABLE IF EXISTS construction_types;

-- code for table creation, below ...
```

{% endshow_solution %}

#### Foreign Keys

Notice how we have made `zoning_type_id` and `construction_type_id` foreign key columns on the `accounts` table.
Integer values will populate these columns, and these values will reference the primary keys of records on their respective tables.
Notice how the 3rd accounts record points to a zoning record with an id of `2` (Commercial) and a construction record with an id of `3` (Reinforced Concrete).

```no-highlight
accounts
id | name             | zoning_type_id | construction_type_id
-------------------------------------------------------------
1  | Ms. Jerrod Swift | 1              | 1
2  | Birdie Nikolaus  | 1              | 2
3  | Mrs. Heath Bosco | 2              | 3


zoning_types
id | name
----------------
1  | Residential
2  | Commercial


construction_types
id | name
----------------
1  | Masonry
2  | Wood
3  | Reinforced Concrete
```

Notice how we are defining each construction type and each zoning type *exactly* once.
This is contrast to our CSV data, where these data points are defined over and over again.
When the data is stored in this way (reducing redundancies to 0), it is considered *normalized*.
This is highly advantageous since we are defining data points *in one place*, and we are saving on disk space at the same time.
If we decide to change these zoning and construction types later on, we only need to modify them in one place.
This is both convenient and ensures that our data is more accurate.

__Note:__ Remember that we don't need to manage the creation of our primary keys.
As long as we `SERIAL`ize the primary key column when creating a table, the creation of primary keys for new records will be done for us!

#### Transfer Data with Ruby

Once your schema is imported, now comes the time to transfer data from the CSV file to your database.
We will use the provided `import.rb` file to accomplish this task.
Your `accounts` table will have many records, but your other tables will have few, one for each *unique* zoning and construction type.

First, let's read our CSV file:

```ruby
csv_records = CSV.readlines('data.csv', headers: true)
```

When reading data with the Ruby CSV library this way, `csv_records` will contain an array of CSV::Row objects.

```no-highlight
$ ruby import.rb

From: /Users/rd/challenges/building-a-database/import.rb @ line 15 :

    10:     connection.close
    11:   end
    12: end
    13:
    14: csv_records = CSV.readlines('data.csv', headers: true)
 => 15: binding.pry

[1] pry(main)> csv_records
=> #<CSV::Table mode:col_or_row row_count:198>
[2] pry(main)> csv_records[0]
=> #<CSV::Row "name":"Ms. Jerrod Swift" "zoning_type":"Residential" "construction_type":"Masonry">
```

We can treat each `CSV::Row` object like a Ruby `Hash`.

```no-highlight
[3] pry(main)> csv_records[0]["name"]
=> "Ms. Jerrod Swift"
```

#### Breaking Down the Problem: Inserting Zoning Types

Right now, we have a large CSV with **a lot** of data. **What we want** is to
take this data, and put it into three different database tables. The `zoning_types`
and `construction_types` tables are rather simple. They simply contain the names
of the zoning types and construction types available. So, let's focus on populating
those, first.

Using Ruby, and our `csv_records` variable, **retrieve a list of unique zoning types**.
Remember, `csv_records` is simply an array of hash-like objects.

{% show_hint %}
Use the Ruby `.map` and `.uniq` methods to filter the `csv_records` array.
{% endshow_hint %}

{% show_solution %}
```ruby
zoning_types = csv_records.map { |record| record["zoning_type"] }  # get all zoning_types
zoning_types.uniq!  # remove duplicates
```
{% endshow_solution %}

Next, iterate over the list of zoning types and insert them into the database.

{% show_solution %}
```ruby
zoning_types.each do |type|
  sql = "INSERT INTO zoning_types (name) VALUES ($1)"
  db_connection do |conn|
    conn.exec_params(sql, [type])
  end
end
```
{% endshow_solution %}

#### Inserting Construction Types

Now, using a similar approach as for inserting zoning types, insert the
construction types listed in the CSV.

{% show_solution %}
```ruby
construction_types = csv_records.map { |record| record["construction_type"] }.uniq

construction_types.each do |type|
  sql = "INSERT INTO construction_types (name) VALUES ($1)"
  db_connection do |conn|
    conn.exec_params(sql, [type])
  end
end
```
{% endshow_solution %}

#### Checking Your Work

If we have done things correctly, we should have populated two of our tables
with unique data. Open up the `psql` console to check your work.

```no-highlight
$ psql building_database

building_database=# SELECT * FROM zoning_types;
 id |    name
----+-------------
  1 | Residential
  2 | Commercial
(2 rows)

building_database=# SELECT * FROM construction_types;
 id |        name
----+---------------------
  1 | Masonry
  2 | Wood
  3 | Reinforced Concrete
  4 | Reinforced Masonry
(4 rows)
```

#### A Problem Arises

If we run our `import.rb` code multiple times, duplicate records will be entered
into our database, thus ruining our goals of normalized data.

Instead of just blindly inserting records, we should first check if the record
already exists. We can do this with a little SQL and Ruby code.

Refactor your code to prevent the creation of duplicate records in the `zoning_types`
and `construction_types` tables.

{% show_hint %}
1. Use a `SELECT` statement to attempt to retrieve the zoning type.
2. If we get back an empty array of results from `conn.exec_params`, then we're safe to insert that zoning type.
{% endshow_hint %}

{% show_solution %}
```ruby
zoning_types.each do |type|
  db_connection do |conn|
    result = conn.exec_params(
      'SELECT name FROM zoning_types WHERE name=$1',
      [type]
    )

    if result.to_a.empty?
      sql = "INSERT INTO zoning_types (name) VALUES ($1)"
      conn.exec_params(sql, [type])
    end
  end
end
```

A similar solution for construction types can be created by replacing the variable
and table names.

{% endshow_solution %}

#### Dependencies

Now that we have the `zoning_types` and `construction_types` table squared away, we are going to now work on inserting a record into the `accounts` table. Why are we dealing with the `accounts` table last? Since the insertion of an account record requires the inclusion of foreign key values from the other two tables in this lesson, this tables records are *dependent* on another table's records. A `zoning_types` record can exist on it's own, but an `accounts` record relies on the presence of a `zoning_types` and `construction_types` record. This is why we've waited until this point to take care of this task.

There are a couple steps that we need to take before actually inserting the `accounts` record.

We must first retrieve the id of the zoning type we just inserted in the above step.
Since we are taking a look at a single CSV record thanks to the iteration we've accomplished in the first step, we can use the zoning type on the CSV record to look up the id. We want to find a record's id _WHERE_ the CSV zoning_type is equal to the name column on zoning_types. Go ahead and write the SQL query that accomplishes this.

{% show_solution %}
```ruby
db_connection do |conn|
  # iterate over CSV data
  csv_records.each do |record|
    zoning_type = record["zoning_type"]
    zoning_type_id = conn.exec_params(
      'SELECT id FROM zoning_types WHERE name=$1',
      [zoning_type]
    )[0]["id"]
  end
end
```
{% endshow_solution %}

Repeat this idea and write a sql query that retrieves the id of the `construction_types` record.

{% show_solution %}
```ruby
db_connection do |conn|
  # iterate over CSV data
  csv_records.each do |record|
    zoning_type = record["zoning_type"]
    construction_type = record["construction_type"]

    # grab the correct foreign key values from each table
    zoning_type_id = conn.exec_params(
      'SELECT id FROM zoning_types WHERE name=$1',
      [zoning_type]
    )[0]["id"]

    construction_type_id = conn.exec_params(
      'SELECT id FROM construction_types WHERE name=$1',
      [construction_type]
    )[0]["id"]
  end
end
```

{% endshow_solution %}

Now that we have the two ids that we need, go ahead and write a final SQL query that will insert these two ids, in addition to the person's name.

{% show_solution %}

```ruby
db_connection do |conn|
  # iterate over CSV data
  csv_records.each do |record|
    zoning_type = record["zoning_type"]
    construction_type = record["construction_type"]

    # grab the correct foreign key values from each table
    zoning_type_id = conn.exec_params(
      'SELECT id FROM zoning_types WHERE name=$1',
      [zoning_type]
    )[0]["id"]

    construction_type_id = conn.exec_params(
      'SELECT id FROM construction_types WHERE name=$1',
      [construction_type]
    )[0]["id"]

    # insert new account record
    name = record["name"]
    conn.exec_params(
      'INSERT INTO accounts (name, zoning_type_id, construction_type_id) VALUES($1, $2, $3)',
      [name, zoning_type_id, construction_type_id]
    )
  end
end
```

{% endshow_solution %}

#### Conclusion

You can verify your work by going into `psql` itself and performing whatever SQL query you would like on your database:

```no-highlight
$ psql building-database
building-database=# SELECT * FROM accounts;
```

When you are done, you should have non-duplicated data in your database. Do not worry about verifying if account names are unique. They are already unique. Congrats! You have successfully normalized this set of data in a relational database!
