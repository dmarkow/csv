---
layout: default
title: CSV document Reader connection
---

# Reader Connection

~~~php
<?php

class Reader extends AbstractCsv implements IteratorAggregate
{
    public function fetchDelimitersOccurrence(array $delimiters, int $nb_records = 1): array
    public function getHeader(): array
    public function getHeaderOffset(): int|null
    public function getIterator(): Iterator
    public function getRecords(): Iterator
    public function setHeaderOffset(?int $offset): self
    public function select(Statement $stmt): ResultSet
    public function supportsHeaderAsRecordKeys(): bool
}
~~~

The `League\Csv\Reader` class extends the general connections [capabilities](/9.0/connections/) to ease selecting and manipulating CSV document records.

## CSV example

Many examples in this reference require an CSV file. We will use the following file `file.csv` containing the following data:

    "First Name","Last Name",E-mail
    john,doe,john.doe@example.com
    jane,doe,jane.doe@example.com
    john,john,john.john@example.com

## Detecting the delimiter character

This method allow you to find the occurences of some delimiters in a given CSV object.

~~~php
<?php

public Reader::fetchDelimitersOccurrence(array $delimiters, int $nb_records = 1): array
~~~

The method takes two arguments:

* an array containing the delimiters to check;
* an integer which represents the number of CSV records to scan (default to `1`);

~~~php
<?php

use League\Csv\Reader;

$reader = Reader::createFromPath('/path/to/file.csv');
$reader->setEnclosure('"');
$reader->setEscape('\\');

$delimiters_list = $reader->fetchDelimitersOccurrence([' ', '|'], 10);
// $delimiters_list can be the following
// [
//     '|' => 20,
//     ' ' => 0,
// ]
// This seems to be a consistent CSV with:
// - the delimiter "|" appearing 20 times in the 10 first records
// - the delimiter " " never appearing
~~~

<p class="message-warning"><strong>Warning:</strong> This method only test the delimiters you gave it.</p>

## Header detection

You can set and retrieve the header offset as well as its corresponding record.

### Description

~~~php
<?php

public Reader::setHeaderOffset(?int $offset): self
public Reader::getHeaderOffset(void): int|null
public Reader::getHeader(void): array
~~~

### Example

~~~php
<?php

use League\Csv\Reader;

$csv = Reader::createFromPath('/path/to/file.csv');
$csv->setHeaderOffset(0);
$header_offset = $csv->getHeaderOffset(); //returns 0
$header = $csv->getHeader(); //returns ['First Name', 'Last Name', 'E-mail']
~~~

### Notes

If no header offset is set:

- `Reader::getHeader` method will return an empty array.
- `Reader::getHeaderOffset` will return `null`.

<p class="message-info">By default no header offset is set.</p>

<p class="message-warning">Because the header is lazy loaded, if you provide a positive offset for an invalid record a <code>RuntimeException</code> will be triggered when trying to access the invalid record.</p>

~~~php
<?php

use League\Csv\Reader;

$csv = Reader::createFromPath('/path/to/file.csv');
$csv->setHeaderOffset(1000); //valid offset but the CSV does not contain 1000 records
$header_offset = $csv->getHeaderOffset(); //returns 1000
$header = $csv->getHeader(); //triggers a Exception
~~~

## Accessing CSV records

### Basic usage

~~~php
<?php

public Reader::getIterator(void): Iterator
public Reader::getRecords(void): Iterator
public Reader::supportsHeaderAsRecordKeys(): bool
~~~

The `Reader` class let's you access all its records using the `Reader::getRecords` method. The method which accepts no argument, returns an `Iterator` containing all CSV document records. It will also:

- Filter out the empty lines;
- Remove the BOM sequence if present;
- Apply the stream filters if supplied;
- Extract the records using the CSV controls characters;

~~~php
<?php

use League\Csv\Reader;

$reader = Reader::createFromPath('/path/to/my/file.csv');
$records = $reader->getRecords();
foreach ($records as $offset => $record) {
    //$offset : represents the record offset
    //var_export($record) returns something like
    // array(
    //  'john',
    //  'doe',
    //  'john.doe@example.com'
    // );
    //
}
~~~

### Usage with a specified header

If a header offset was specified using the `setHeaderOffset` method

- The found header record is:
    - combined to each CSV record to return an associated array whose keys are composed of the header values.
    - removed from the returned iterator.
- The returned records are normalized to the number of fields contained in the header record
    - Missing fields are added with `null` content.
    - Extra fields are truncated.

~~~php
<?php

use League\Csv\Reader;

$reader = Reader::createFromPath('/path/to/my/file.csv');
$reader->setHeaderOffset(0);
$records = $reader->getRecords();
foreach ($records as $offset => $record) {
    //$offset : represents the record offset
    //var_export($record) returns something like
    // array(
    //  'Fist Name' => 'john',
    //  'Last Name' => 'doe',
    //  'E-mail' => john.doe@example.com'
    // );
    //
}
~~~

<p class="message-warning">If the header record contains non unique values, a <code>RuntimeException</code> exception is triggered </p>


You can avoid this exception by using the `Reader::supportsHeaderAsRecordKeys` method. The method returns `true` if `Reader::getHeader` returns:

- an empty record;
- or, a record containing only unique string values;

~~~php
<?php

use League\Csv\Reader;

$reader = Reader::createFromPath('/path/to/my/file.csv');
$reader->setHeaderOffset(0);
//var_export($reader->getHeader()) returns something like
// array(
//  'First Name',
//  'Last Name',
//  'E-mail'
// );
$reader->supportsHeaderAsRecordKeys(); //return true;
$reader->getRecords(); //returns an Iterator

$reader->setHeaderOffset(3);
//var_export($reader->getHeader()) returns something like
// array(
//  'john',
//  'john',
//  'john.john@example.com'
// );
$reader->supportsHeaderAsRecordKeys(); //return false;
$reader->getRecords(); //throws a RuntimeException
~~~

### Improving iteration

Because the `Reader` class implements the `IteratorAggregate` interface you can directly iterate over each record using the `foreach` construct and an instantiated `Reader` object. You will get the same results as if you had called `Reader::getRecords`.

~~~php
<?php

use League\Csv\Reader;

$reader = Reader::createFromPath('/path/to/my/file.csv');
$reader->setHeaderOffset(0);
foreach ($reader as $offset => $record) {
    //$offset : represents the record offset
    //var_export($record) returns something like
    // array(
    //  'Fist Name' => 'john',
    //  'Last Name' => 'doe',
    //  'E-mail' => john.doe@example.com'
    // );
    //
}
~~~

## Selecting CSV records

### Description

~~~php
<?php

public Reader::select(Statement $stmt): ResultSet
~~~

If you require a more advance record selection, you may use the `Reader::select` method. This method uses a [Statement](/9.0/reader/statement/) object to process the `Reader` object. The found records are returned as a [ResultSet](/9.0/reader/resultset) object.

### Example

~~~php
<?php

use League\Csv\Reader;
use League\Csv\Statement;

$reader = Reader::createFromPath('/path/to/my/file.csv');
$stmt = (new Statement())
    ->offset(3)
    ->limit(5)
;
$records = $reader->select($stmt);
//$records is a League\Csv\ResultSet object
~~~

<p class="message-info">This method is equivalent of <a href="/9.0/reader/statement/#apply-the-constraints-to-a-csv-document">Statement::process</a>.</p>