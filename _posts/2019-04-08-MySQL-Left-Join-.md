---
layout: post
title:  MySQL LEFT JOIN
categories: MySQL LEFT-JOIN
tags:  MySQL LEFT-JOIN
author: wenzhilee77
---

# Introduction to MySQL LEFT JOIN

The LEFT JOIN allows you to query data from two or more tables. Similar to the INNER JOIN clause, the LEFT JOIN is an optional clause of the SELECT statement, which appears immediately after the FROM clause.

Suppose that you want to join two tables t1 and t2.

The following statement shows how to use the LEFT JOIN clause to join the two tables:

```mysql
SELECT
    select_list
FROM
    t1
LEFT JOIN t2 ON
    join_condition;
```

When you use the LEFT JOIN clause, the concepts of the left table and the right table are introduced.

In the above syntax, t1 is the left table and t2 is the right table.

The LEFT JOIN clause selects data starting from the left table (t1). It matches each row from the left table (t1) with every row from the right table(t2) based on the join_condition.

If the rows from both tables cause the join condition evaluates to TRUE, the LEFT JOIN combine columns of rows from both tables to a new row and includes this new row in the result rows.

In case the row from the left table (t1) does not match with any row from the right table(t2), the LEFT JOIN still combines columns of rows from both tables into a new row and include the new row in the result rows. However, it uses NULL for all the columns of the row from the right table.

In other words, LEFT JOIN returns all rows from the left table regardless of whether a row from the left table has a matching row from the right table or not. If there is no match, the columns of the row from the right table will contain NULL.

The following Venn diagram helps you visualize how the LEFT JOIN clause works. The intersection between two circles are rows that match in both tables, and the remaining part of the left circle are rows in the t1 table that do not have any matching row in the t2 table. Hence, all rows in the left table are included in the result set.

![](/images/join/001.png)

Notice that the returned rows must also match the conditions in the WHERE and  HAVING clauses if those clauses are available in the query.

# MySQL LEFT JOIN examples

## Using MySQL LEFT JOIN clause to join two tables

Let’s take a look at the customers and orders tables in the sample database.

![](/images/join/002.png)

In the database diagram above:
* Each order in the orders table must belong to a customer in the customers table.
* Each customer in the customers table can have zero or more orders in the orders table.

To find all orders that belong to each customer, you can use the LEFT JOIN clause as follows:

```mysql
SELECT 
    c.customerNumber, c.customerName, orderNumber, o.status
FROM
    customers c
        LEFT JOIN
    orders o ON c.customerNumber = o.customerNumber;
```

![](/images/join/003.png)

The left table is customers, therefore, all customers are included in the result set. However, there are rows in the result set that have customer data but no order data e.g. 168, 169, etc. The order data in these rows are NULL. It means that these customers do not have any order in the orders table.

Because we used the same column name ( customerNumber) for joining two tables, we can make the query shorter by using the following syntax:

```mysql
SELECT
    customerNumber,
    customerName,
    orderNumber,
    status
FROM
    customers
LEFT JOIN orders USING (customerNumber);
```

If you replace the LEFT JOIN clause by the INNER JOIN clause, you get the only customers who have placed at least one order.

## Using MySQL LEFT JOIN clause to find unmatched rows

The LEFT JOIN clause is very useful when you want to find the rows in the left table that do not match with the rows in the right table. To find the unmatching rows between two tables, you add a WHERE clause to the SELECT statement to query only rows whose column values in the right table contains the NULL values.

For example, to find all customers who have not placed any order, you use the following query:

```mysql
SELECT 
    c.customerNumber, c.customerName, orderNumber, o.status
FROM
    customers c
        LEFT JOIN
    orders o ON c.customerNumber = o.customerNumber
WHERE
    orderNumber IS NULL;
```

![](/images/join/004.png)

## Condition in WHERE clause vs. ON clause

See the following example.

In this example, we used the LEFT JOIN clause to query data from the  orders and  orderDetails tables. The query returns an order and its detail, if any, for the order 10123.

![](/images/join/005.png)

However, if you move the condition from the WHERE clause to the ON clause:

```mysql
SELECT
    o.orderNumber,
    customerNumber,
    productCode
FROM
    orders o
LEFT JOIN orderDetails d
    ON o.orderNumber = d.orderNumber AND
       o.orderNumber = 10123;
```

It will have a different meaning.

In this case, the query returns all orders but only the order 10123 will have detail associated with it as shown below.

![](/images/join/006.png)

Notice that for INNER JOIN clause, the condition in the ON clause is equivalent to the condition in the WHERE clause.


# 参考

https://www.mysqltutorial.org/mysql-left-join.aspx