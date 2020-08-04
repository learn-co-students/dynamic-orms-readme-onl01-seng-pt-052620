# Dynamic ORMs

## Objectives

1. Explain why a dynamic ORM is useful to us as developers
2. Build a basic dynamic ORM
3. Build a dynamic ORM that can be used by any given Ruby class

## Why Dynamic ORMs?

As developers, we understand the need for our Ruby programs to be able to connect with a database. 
Any complex application is going to need to persist some data. 
Also we recognize the need for the connection between our program and our DB to be organized and sensible. 
That is why we use the ORM Design Pattern: 
  Ruby class is mapped to a database table. 
  Instances of that class are represented as rows in that table.

We can implement this mapping by using a class to create a database table:

```ruby
class Song
  attr_accessor :name, :album 
  attr_reader :id 

  def initialize(id=nil, name, album)
    @id = id 
    @name = name 
    @album = album 
  end

  def self.create_table
    sql = "CREATE TABLE IF NOT EXISTS songs(
      id INTEGER PRIMARY KEY, 
      name TEXT, 
      album TEXT
    )" 
    DB[:conn].execute(sql)
  end 
end 

```
We have created our `songs` table, for our `Song` class. 
The column names for the table are taken from `attr_accessor`s of the `Song` class. 

# LIMITATIONS 
Our `.create_table` method is dependent on knowing exactly what to name our table and columns.
Every class in our program requires us to re-write this method, swapping out different table and column names each time. 
With a Dynamic ORM, we can abstract all of our conventional ORM methods into flexible, abstract and shareable methods.

# What is a Dynamic ORM? 
A Dynamic ORM allows us to map an existing DB Table to a class 
It allows us to write methods that can use nothing more than information regarding a specific DB table to: 
  
  * Create `attr_accessors` for a Ruby Class
  * Create shareable methods for Inserting, Updating, Selecting and Deleting Data from the DB Table

- Pattern 
  First creating the DB Table and having your program do all the work of writing your ORM methods for you, based on that table
  This is exactly how we will develop Web Applications in Sinatra and Rails. 

# Creating Our ORM 

Writing an ORM is hard. 

# STEP 1: Setting Up The Dababase 

To create a Dynamic ORM, we start by Creating our Database and Songs Table 

`config/environment.rb` 

```ruby 

require 'sqlite3'

DB = {:conn => SQLite3::Database.new("db/songs.db")}
DB[:conn].execute("DROP TABLE IF EXISTS songs")

sql = "CREATE TABLE IF NOT EXISTS songs(
  id INTEGER PRIMARY KEY, 
  name TEXT, 
  album TEXT" 
); 

DB[:conn].execute(sql)
DB[:conn].results_as_hash = true 

``` 
# Explanation 

1. Creating the DB 
2. Drop `songs` table if it exists to avoid any errors 
3. Creating the `songs` table. 

We use the `#results_as_hash` method, available to use from the SQLite3-Ruby gem. 
This methods says: 

When a `SELECT` method is executed, don't return a database row as an array, return it as a hash with the Column Names as Keys. 


DB[:conn].execute("SELECT * FROM songs LIMIT 1")  


```ruby
#Array Version 
[[1, "Hello", "25"]]
```

```ruby
#Hash Version 
{
  "id"=>1, 
  "name"=>"Hello", 
  "album"=>"25", 
  0 => 1, 
  1 => "Hello", 
  2 => "25"
}
```
 
# Step 2: Builing  'attr_accessor's  From Column Names'

1. Use the Column Names of the `songs` table to dynamically create the `attr_accessor`s of our `Song` class 
2. We first need to collect the Column Names from our `songs` table. 
3. In order to collect the column names from the songs table we need to tell our `Song` class what table to query. 
4. We DO NOT want to tell the `Song` class to query the `songs` table explicitly. This would not be flexible. 
5. If we defined a method that explicitly references the `songs` table, we would not be able to extract that method into a `shareable method` later on.

The goal of our Dynamic ORM is to defined a series of methods that can be shared by ANY CLASS 
We need to avoid explicitly referencing Table and Column Names. 


# The #table_name Method 
```ruby

class Song
  def self.table_name
    self.to_s.downcase.pluralize 
  end 
end 
```

# Notes 

This method takes the name of the class, referenced by the `self` keyword. 
It turns it into a string with `#to_s`, downcases / un-capitalizes that string and then "pluralizes" it.

The `#pluralize` method is provided to us by the `active_support/inflector` code library, required at the top of `lib/song.rb`.


We have a method that grabs us the table name we want to query for column names
Now we build a method that actually grabs us those column names.


# The #column_names Method  
# Querying a Table For Column Names

* This query provides us the names of a table's columns 

```sql
PRAGMA table_info(<table name>)
```

# Notes 
This utilizes <a href="#resources">PRAGMA</a> 

It return us (thanks to `#results_as_hash` method) an array of hashes describing the table itself. 
Each hash will contain information about one column. 
The array of hashes will look something like this:

```ruby
 [
   {
     "cid"=>0,
     "name"=>"id",
     "type"=>"INTEGER",
     "notnull"=>0,
     "dflt_value"=>nil,
     "pk"=>1,
     0=>0,
     1=>"id",
     2=>"INTEGER",
     3=>0,
     4=>nil,
     5=>1
     },
  {
    "cid"=>1,
    "name"=>"name",
    "type"=>"TEXT",
    "notnull"=>0,
    "dflt_value"=>nil,
    "pk"=>0,
    0=>1,
    1=>"name",
    2=>"TEXT",
    3=>0,
    4=>nil,
    5=>0
    },
  {
    "cid"=>2,
    "name"=>"album",
    "type"=>"TEXT",
    "notnull"=>0,
    "dflt_value"=>nil,
    "pk"=>0,
    0=>2,
    1=>"album",
    2=>"TEXT",
    3=>0,
    4=>nil,
    5=>0
    }
]
```
The only thing we need to grab out of this Hash is the name of each column. 
Each hash has a `"name"` key that points to a value of the column name.

# Building Our Method:

```ruby
def self.column_names
  DB[:conn].results_as_hash = true 
  sql = "PRAGMA table_info('#{table_name}')"

  table_info = DB[:conn].execute(sql)
  column_names = []
  
  table_info.each do |column|
  column_names << column["name"]
  end
  column_names.compact   #compact removes nil elements 
end 
```

# Notes
We write a SQL statement using the `pragma` keyword and the `#table_name` method 
The `#table_name` helps us access the name of the table we are querying.  
We iterate over the resulting array of hashes to collect `just the name of each column`. 
We call `#compact` on that just to be safe and get rid of any `nil` values that may end up in our collection.

The return value of calling `Song.column_names` will therefore be:
```ruby
["id", "name", "album"]  
```

Now we have a method that returns us an array of column names.
We can use this collection to create the `attr_accessors` of our `Song` class. 



### Metaprogramming Our attr_accessor`s
We can tell our `Song` class that it should have an `attr_accessor` named after each column name with the following code:

```ruby
class Song 
  def self.table
    #table_name code 
  end 

  def self.column_names
    #column_names code 
  end 

  self.column_names.each do |column_name|
    attr_accessor column_name.to_sym
    end
end

```
# Notes 

We iterate over the column names stored in the `column_names` class method 
Then we set an `attr_accessor` for each one
We make sure to convert the column name string into a symbol with the `#to_sym` method
We do this because `attr_accessor`s must be named with symbols.

This is metaprogramming because we are writing code that writes code for us. 

When we set the `attr_accessor`s this way 
 - A reader and writer method for each column name is dynamically created
 - This is done without us ever having to explicitly name each of these methods.

## Step 3: Building An Abstract `#initialize` Method

Now that our `attr_accessor`s are defined, we can build the `#initialize` method for the `Song` class. 
With our  Dynamic ORM, we want our `#initialize` method to be abstract and not specific to the `Song` class. 
We want to be able to remove it into a parent class that any other class can inherit from. 
Once again, we'll use metaprogramming to achieve this.


* We Want To Be Able To Create A New Song Like This

```ruby
song = Song.new(name: "Hello", album: "25")

song.name
# => "Hello"

song.album
# => "25"
```
# Notes 

We need to define our `#initialize` method to take in a hash of named, or keyword, arguments. 
We `DO NOT` want to explicitly name those arguments. 

Here's how we can do it:

# The #initialize Method

```ruby

def initialize(options={})
  options.each do |property, value|
    self.send("#{property}=", value)
    end 
end 

```
# Notes 

We define our method to take in an argument of `options`, which defaults to an empty hash. 
We expect `#new` to be called with a hash
When we refer to `options` inside the `#initialize` method, we expect to be operating on a hash.

We iterate over the `options` hash and use metaprogramming `#send` method 
We interpolate the name of each hash key as a method that we set equal to that key's value. 
As long as each `property` has a corresponding `attr_accessor`, this `#initialize` method will work.


# Step 4: Writing Our ORM Methods

Writing conventional ORM methods, like `#save` and `#find_by_name`, in a dynamic fashion. 
They will be abstract and not specific to the `Song` class.
We can later extract these methods and share them among any number of classes.


# Saving Records in a Dynamic Manner

This is basic SQL statement required to save a given song record:

```sql
INSERT INTO songs (name, album) VALUES 'Hello', '25';
```

# Notes 
In order to write a method that can `INSERT` any record to any table
We need to be able to craft the above SQL statement without explicitly referencing the `songs` table 
or column names and without explicitly referencing the values of a given `Song` instance.

# Abstracting the Table Name
We have a method that provides us the table name that is associated with any given class 
`<class name>.table_name`

The conventional `#save` is an *instance* method. 
So, inside a `#save` method, `self` will refer to the instance of the class, not the class itself. 
In order to use a class method inside an instance method, we need to do the following:

```ruby
def some_instance_method
  self.class.some_class_method 
end
```

# Note 

To access the table name we want to `INSERT` into from inside our `#save` method
we will use the following:

```ruby
self.class.table_name
```

# Note 
We can wrap up this code in a handy method, `#table_name_for_insert`

```ruby
def table_name_for_insert
  self.class.table_name 
end 
```

Now let's grab our column names in an abstract manner.

# Abstracting the Column Names

We already have a method for grabbing the column names of the table associated with a given class:

```ruby
self.class.column_names
```

In the `Song` class, this will return:

```ruby
["id", "name", "album"]
```

# Problem 

When we `INSERT` a row into a database table for the first time, we DO NOT `INSERT` the `id` attribute. 
Our Ruby object has an `id` of `nil` before it is inserted into the table. 

Our SQL database handles the creation of an ID for a given table row 
Then we will use that ID to assign a value to the original object's `id` attribute.

* When we `save` our Ruby object, we should not: 

Include the id column name or insert a value for the id column. 
Therefore, we need to remove `"id"` from the array of column names returned from the method call above:

```ruby
self.class.column_names.delete_if {|col| == "id"|} 

["name", "album"]   # Returned Result 


```
# Column Names Needed To Craft Our `INSERT` Statement. 

What the statement needs to look like:

```sql
INSERT INTO songs (name, album)  VALUES 'Hello', '25';
```

# Notes 
Notice that the column names in the statement are comma separated. 
Our column names returned by the code above are in an array. 
Let's turn them into a comma separated list, contained in a string:

```ruby
self.class.column_names.delete_if{|column| column == "id"}.join(", ")

"name, album"  # Returned Result  
```

# Notes 
Now we need to grab a comma separated list of the column names of the table associated with any given class.

We use this method to wrap our code, `#col_names_for_insert`

```ruby
def col_names_for_insert 
  self.class.column_names.delete_if{|column| column== "id"}.join(", ")
end 
```

# Grabbing The Values We Want To INSERT 

When inserting a row into our tableWe grab the values to insert by grabbing the values of that instance's `attr_reader`s. 
How can we grab these values without calling the reader methods by name?

The names of that `attr_accessor` methods were derived from the column names of the table associated to our class. 
Those column names are stored in the `#column_names` class method.

We know how to programmatically invoke a method, without knowing the exact name of the method, using the `#send` method.

We iterate over the column names stored in `#column_names` 
Then we use the `#send` method with each individual column name to invoke the method by that same name and capture the return value:

```ruby
values = [] 
self.class.column_names.each do |column_name|
  values << "'#{send(column_name)}'" unless send(column_name).nil? 
  end
  valies = []
end  
```
# Notes 
We push the return value of invoking a method via the `#send` method, unless that value is `nil` 
It would be `nil` for the `id` method before a record is saved


We are wrapping the return value in a string because we are trying to craft a string of SQL.  
Each individual value will be enclosed in single quotes, `' '`, inside that string. 
That is because the final SQL string will need to look like this:

```sql
INSERT INTO songs (name, album) VALUES 'Hello', '25'; 
```
SQL expects us to pass in each column value in Single Quotes. 
The above code however, will result in a `values` array. 
SQL expects us to pass in each column value in single quotes.

```ruby
["'The Name Of The Song'", "'The Album Of The Song'"]
```

We need comma separated values for our SQL statement. 
Let's join this array into a string:

```ruby
values.join(", ")
```

We wrap up this code using this method `#values_for_insert`

```ruby
def values_for_insert 
  values = [] 
  self.class.column_names.each do |column_name|
    values << "'#{send(column_name)}'" unless send(column_name).nil? 
  end 
  values.join(", ")
end 
```


Now we have an abstract, flexible way to grab each of the constituent parts of the SQL statement to save a record
We will put them all together into the `#save` method:

# The #save Method

```ruby
def save
  DB[:conn].execute("INSERT INTO #{table_name_for_insert} (#{col_names_for_insert}) VALUES (?)", [values_for_insert])
  @id = DB[:conn].execute("SELECT last_insert_rowid() FROM #{table_name_for_insert}")[0][0]
end 
```
# Note 
Using `String` interpolation for an SQL query creates a SQL injection vulnerability. 
We have previously stated is a bad idea as it creates a security issue, however, we're using these examples to illustrate how dynamic ORMs work.

# Selecting Records in a Dynamic Manner
Now we understand how our dynamic, abstract, ORM works
Let's build the `#find_by_name` method.

```ruby
def self.find_by_name(name)
  DB[:conn].execute("SELECT * FROM #{self.table_name} WHERE name = ?", [name])
end
```
# Note 

This method is dynamic and abstract because it does not reference the table name explicitly. 
Instead it uses the `#table_name` class method we built that will return the table name associated with any given class.

## Conclusion

Remember, dynamic ORMs are hard. Spend some time reading over the code in `lib/song.rb` and playing with the code in `bin/run`. Practice creating, saving and querying songs in the `bin/run` file and run the program again and again until you get a better feel for it. 

Now that we have all of these great dynamic, abstract methods that connect a class to a database table, we'll move on to extracting into a parent class that any other class can inherit from.

## Resources
<a name="pragma"></a>
[SQLite- PRAGMA](http://www.tutorialspoint.com/sqlite/sqlite_pragma.htm)

[PRAGMA](https://www.sqlite.org/pragma.html#pragma_table_info)

<p data-visibility='hidden'>View <a href='https://learn.co/lessons/dynamic-orms-readme'>Dynamic ORMs</a> on Learn.co and start learning to code for free.</p>

<p class='util--hide'>View <a href='https://learn.co/lessons/dynamic-orms-readme'>Dynamic ORMs</a> on Learn.co and start learning to code for free.</p>
