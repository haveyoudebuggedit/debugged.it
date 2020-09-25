---
slug: datastructures-101-basics
guid: https://pasztor.at/blog/datastructures-101-basics/
authors:
- janos
categories:
- Technology
date: "2016-11-23T00:00:00Z"
publishDate: "2016-11-23T00:00:00Z"
summary: Why would you learn data structures? You won't need it unless you are a programmer,
  database engineer or university student... wait, you are? Never mind, keep on reading.
tags:
- Software Development
title: 'Datastructures 101: Basics'
---

## Why bother?

Why do you need to learn data structures at all? After all, can't the computer take care of storing all your data? As
it turns out, the answer isn't that simple. Depending on what kind of data you want to store, the data structure 
has to be adapted.

Let's imagine a simple example: you want to store a bunch of records in memory or on disk. Since you want a simple solution, you just store the data as it is. Something like this:

![](posts/datastructures-101-basics/table.svg)

As you can see, the data is ordered by numeric ID, so searching by name is going to require you to read the entire 
dataset. For simplicity's sake let's just say, for four records it is going to take up to four operations to find a 
given record. For `n` records, this data structure would take *up to* `n` steps to complete.

This kind of limiting behavior is usually described with the
[Big O notation](https://en.wikipedia.org/wiki/Big_O_notation). Bear with me; we are going to do a tiny bit of math 
here.

Let's imagine an algorithm that for any <code>n</code> input length needs <code>4n³ + 10n² + n + 5</code> operations to 
complete. In the Big O notation, we would describe this algorithm as being <code>O(n³)</code>. If we take a large 
enough data set, the most significant factor of n will dominate the execution time. This means that we can use that to
classify the algorithm's estimated speed on large databases. That is the Big O notation:
<code>O(n<sup>largest_factor</sup></code>). In this example it's <code>O(n³)</code>. (You can breathe out now, that's
all the math we are going to do today.) If you want to read more on the Big O, I would recommend <a href="http://web.mit.edu/16.070/www/lecture/big_o.pdf">this paper</a> 

So our data storage above could be described as `O(n)` for searching by name. If you are building a database system 
with this kind of data storage, you will need a *full table scan* to find a given record. While this is something you
might want to avoid for larger data structures, it is perfectly fine if you just have a few records because it's not 
worth the effort of making it faster. 

## Linked lists

If you are a C or C++ programmer, you may notice that the data structure above has a huge flaw: you can't easily 
extend it. In modern computers multiple programs run at once, so you can't be sure if the next block is freely 
available. It may belong to a different program, so you can't just go ahead and write to it. In other words, you would
have to copy over all the data into a newly allocated region, which can be quite slow.

The simplest data structure to solve this problem is the **linked list**. It looks like this:

![](posts/datastructures-101-basics/linked-list.svg)

As you can see, we now have a separate memory space for every record and every record only contains the memory 
address of the next record (a pointer). This allows us to attach new records at will, without needing to copy over 
everything. We can even create a doubly linked list if we want to traverse the list in both directions.

{{% tip %}}
**Recommended exercise:** to better understand linked lists, implement and test the insert, search and delete operations for linked lists in the language of your choice.
{{% /tip %}}

## Hash tables

Quite unsatisfyingly, the linked lists are still only `O(n)`. They do provide an easily extensible data structure, 
but they do not yield any speed benefits. Let's create a more advanced data structure named *hash tables*.

Hash tables take the value we want to index, apply some transformation to it and then use it to group the records. A very simple hash table would be to take the first letter of every name and group the records by that. Just like this:

![](posts/datastructures-101-basics/hash-table.svg)

The performance of a hash table depends on how *evenly* the hash table is distributed. In other words, all your *hash buckets* (or partitions) would need to contain roughly the same number of items. This hashing algorithm (taking the first letter) obviously does a very poor job because names are usually very unevenly distributed.

Let's look at a slightly better example. Let's create a hash table for numbers and as a hashing algorithm let's use modulo 3. In other words, our hashing algorithm will divide the number by three and take the remainder as a hash value. 

![](posts/datastructures-101-basics/hash-table2.svg)

If we wanted to locate the number 8, we would calculate 8 divided by 3, which is 2 and the remainder is 2. So we look
up the items in bucket 2 and go down in the list until we find the number 8, or we find a number that's larger, 
indicating that 8 is not present.

This takes 4 steps. Why 4? Because we have a total of 9 items, evenly distributed into three buckets, and we need one
extra step for looking up the hash. In other words, an **evenly distributed** hash table will need `n / buckets + 1` 
steps for a lookup, where n is the number of items in the hash table. (This is of course a simplified calculation.) 
Although this is *technically* still `O(n)`, it performs significantly better than the linked list if the number of 
buckets is high enough.

This goes to show that Big O is not everything when comparing algorithms. That's why the `Θ` notation is often 
used to indicate the *average* time the algorithm needs.

## Pro tip: multiple indexing

If you have records with multiple fields, you may want to perform searches on more than one column. Therefore, 
instead of creating one data structure for all your data, you may want to consider creating multiple *indexes* that 
contain the data sorted in different ways. Modern database engines do this to speed up queries.

However, while the searches become faster, inserts get slower as your code (or the database engine) needs to 
update these indexes. In other words, don't go over board with indexing.

## Next up

So far, we have only covered data structures with (mostly) `O(n)` performance for inserts and searches. In the next 
article, we are going to take a look at the more advanced *Binary Search Tree*, and we'll also cover the dreaded 
*Btree* and *Btree+* structures in future.
