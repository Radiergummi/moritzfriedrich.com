---
title: "Internal CLI"
date: 2020-03-17T10:12:39+01:00
draft: false
tags:
    - project
    - messengerpeople
---

As a company grows, its infrastructure grows with it. This makes it more and more unwieldy to manage all tools involved: Status dashboards, the company CRM, cloud
provider control panels and home-grown software are all required to keep the business running, and everything needs to more or less work. Sometimes, there's a
database query nobody ever cared enough about to build an interface for it: Say, archiving a customer, or checking the activity status of a bunch of customers by 
some arcane filter. Inevitably, someone will need a quick insight or a toggle or a new graph somewhere.  
While we do have a more-or-less central place for such tasks, the time required to add new features to it often can't be justified. We end up with someone from the
engineering team repeating the same task manually now and then, without ever realizing the total time budget has long exceeded the original estimate.

-----

Now, as an engineer, I spend most of my time on the command line. It is often the fastest way to achieve things, and the power of unix pipes allows for infinite 
chaining of commands. Using an existing CLI framework, implementing new features is also much faster of a task than having to build a GUI! This lead me to creating
our very own, company-internal CLI application to provide the leanest possible abstraction layer over every single piece of engineer interaction.

As we use PHP for pretty much everything---even [horizontally scalable, threaded workers]({{< ref "massively-parallel-php.md" >}})---it was clear we'd use our own 
flavor of [Symfony Console](https://symfony.com/doc/current/components/console.html) for the job. While the console component is awesome, it is a little verbose for
my taste. As we keep our business logic in [a custom "framework"]({{< ref "application-framework.md" >}}) anyway, it made sense to create a set of decorator 
components that add all helpers we commonly need and make sure dependency injection works properly. For example, Symfony gives us 
[the option to create tables](https://symfony.com/doc/current/components/console/helpers/table.html), like so:
```php
<?php # Opening tag required due to https://github.com/alecthomas/chroma/issues/210
$table = new Table($output);
$table
    ->setHeaders(['ISBN', 'Title', 'Author'])
    ->setRows([
        ['99921-58-10-7', 'Divine Comedy', 'Dante Alighieri'],
        ['9971-5-0210-0', 'A Tale of Two Cities', 'Charles Dickens'],
        ['960-425-059-0', 'The Lord of the Rings', 'J. R. R. Tolkien'],
        ['80-902734-1-6', 'And Then There Were None', 'Agatha Christie'],
    ]);
$table->render();
```
This is fine, but the requirement to specify the table headers makes it a little laborious when working with multi-dimensional data structures. Compare this to our
table wrapper:
```php
<?php # Opening tag required due to https://github.com/alecthomas/chroma/issues/210
$this->table([
    [
        'ISBN' => '99921-58-10-7',
        'Title' => 'Divine Comedy',
        'Author' => 'Dante Alighieri'
    ],
    [
        'ISBN' => '9971-5-0210-0',
        'Title' => 'A Tale of Two Cities',
        'Author' => 'Charles Dickens'
    ],
    [
        'ISBN' => '960-425-059-0',
        'Title' => 'The Lord of the Rings',
        'Author' => 'J. R. R. Tolkien'
    ],
    [
        'ISBN' => '80-902734-1-6',
        'Title' => 'And Then There Were None',
        'Author' => 'Agatha Christie'
    ],
]);
```

This is much better when working with possibly-unknown records, and it also frees us from having to statically define properties we want to see when querying a data
set! Both examples above result in the same table being written.

**What do we need this for, however?** Easy: I built a command line application with the ability to perform CRUD on database resources with the minimum amount of 
code possible, allowing to list, filter, sort, search and paginate records, as well as deleting, updating and creating individual records. Obviously, this is a 
pretty dangerous tool, so it's limited to engineers: The CLI requires a configuration (.env) file in the user home directory with their database credentials. This
ensures only those with actual database access can use the tool. How does this look like in practice, though?

**Listing records:**
```bash
# Show the third page with 10 records per page of customers created since 
# the first of january, 2020 whose name contains "moritz" and sort them 
# by their name ascending, then by creation date descending.
console customers:list \
    --filter created_at>01-01-2020 \
    --filter name:moritz \
    --page 3 \
    --per-page 10 \
    --sort name \
    --sort created_at \
    --descending created_at

# ...or using the shorthands:
console customers:list \
    -f created_at>01-01-2020 \
    -f name:moritz \
    -p 3 \
    -P 10 \
    -s name \
    -s created_at \
    -d created_at
```

**Retrieving individual records:**
```bash
# Find a customer whose UUID contains 3eaa3b18. We need the column/c option because
# it isn't the primary key of the customers table.
console customers:find 3eaa3b18 \
    --column=uuid

# Find a customer by its (primary key) ID:
console customers:find 42223218

# Find a customer by its exact name:
console customers:find "Moritz Friedrich" \
    --column=name \
    --exact
```

**Updating records:**
```bash
# Update a customer identified by their ID
console customers:update 42223218 \
    address="Testlane 123, 74253 Testville" \
    invoice_email="test@example.com"

# Update a customer identified by their exact name
console customers:update "Moritz Friedrich" \
    --column=name \
    --exact \
    address="Testlane 123, 74253 Testville" \
    invoice_email="test@example.com"
```

**Deleting records:**
```bash
# Delete a customer identified by their ID
console customers:delete 42223218

# Delete a customer identified by their exact name
console customers:delete "Moritz Friedrich" \
    --column=name \
    --exact
```

**Creating records:**
```bash
# Create a new customer
console customers:create \
    name="Moritz Friedrich" \
    address="Testlane 123, 74253 Testville" \
    invoice_email="test@example.com"
```

Now, I think that's pretty cool: Imagine writing those queries by hand every time! Let's look behind the scenes, though. The way the console component works is that
you need to register individual command classes that handle those calls, so usually, we'd need to create separate classes here: `ListCustomersCommand`, 
`FindCustomerCommand`, `UpdateCustomerCommand`, `DeleteCustomerCommand` and `CreateCustomerCommand` in this case. Even if those had base classes that did most of 
the work, that'd still be _lots_ of boilerplate. So as a much quicker alternative, I built an abstract `CrudCommandCollection` class which registers all those 
commands automatically upon invocation. This leaves us with the following class, named `CustomerCommandCollection`:
```php
<?php # Opening tag required due to https://github.com/alecthomas/chroma/issues/210
class CustomerCommandCollection extends CrudCommandCollection
{
    protected string $table = 'customers';
}
```
Yep, I'm not kidding. **All the functionality above** and quite a few features I didn't even mention yet are completely covered by the above class! As that single 
property might hint on, things can be customized by setting several properties or implementing protected methods---all of that is optional though, to reduce the 
friction required to support new resources as much as possible.  
Nevertheless, all text printed to the screen, all queries executed against the database, all properties displayed and all data formatted have hooks to be 
overwritten.
