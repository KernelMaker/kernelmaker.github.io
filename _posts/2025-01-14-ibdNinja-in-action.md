---
layout: post
title: 'Dissecting an Extreme Sparse MySQL Table Using ibdNinja'
---

I previously wrote a blog post to introduce the concept of MySQL B+ Tree splits and explained how to deliberately construct a highly sparse table by hacking the current rules. Then in the next post, I introduced **ibdNinja**, a tool I developed to parse the MySQL data file. In this follow-up post, I demonstrate how to use **ibdNinja** to analyze and verify this table's data file.

This post combines two topics:

1. Constructing a highly sparse InnoDB table.  

   *Reference:* [Hack MySQL: How to Create an Extremely Sparse InnoDB Table (B+Tree)](https://kernelmaker.github.io/Hack-InnoDB)

2. Using [ibdNinja](https://github.com/KernelMaker/ibdNinja) to analyze the table's data file and identify interesting findings.

   *Reference:*[ibdNinja: A powerful tool for parsing and analyzing MySQL 8.0 (.ibd) data files](https://kernelmaker.github.io/ibdNinja)

## 1. Constructing a Highly Sparse Table

By running the following SQL file using the `source` command in the MySQL client, you can create a highly sparse InnoDB table called `tbl`:

```
mysql> source /PATH_TO_THIS_FILE/load.sql
```

**load.sql:**

```
-- Create the database and table
CREATE DATABASE IF NOT EXISTS test;
USE test;

DROP TABLE IF EXISTS tbl;
CREATE TABLE tbl (
    pk INT NOT NULL,
    val CHAR(240) NOT NULL DEFAULT 'a',
    PRIMARY KEY (pk)
) ENGINE=InnoDB CHARSET=LATIN1;

-- Create the stored procedure
DELIMITER //

CREATE PROCEDURE PopulateTable()
BEGIN
    DECLARE i INT;

    -- Insert initial values from 1 to 32
    SET i = 1;
    WHILE i <= 32 DO
        INSERT INTO test.tbl (pk, val) VALUES (i, 'a');
        SET i = i + 1;
    END WHILE;

    -- Insert values from 10000001 to 10000027
    SET i = 10000001;
    WHILE i <= 10000027 DO
        INSERT INTO test.tbl (pk, val) VALUES (i, 'a');
        SET i = i + 1;
    END WHILE;
    -- The first leaf page splits into 2 leaf pages:
    -- [1 ... 29],[30, 31, 32, 10000001 ... 10000027]

    -- Insert values from 10000028 to 10000054
    SET i = 10000028;
    WHILE i <= 10000054 DO
        INSERT INTO test.tbl (pk, val) VALUES (i, 'a');
        SET i = i + 1;
    END WHILE;

    -- Insert alternating values starting from 33 to 1000000
    SET i = 33;
    WHILE i < 100000 DO
        INSERT INTO test.tbl (pk, val) VALUES (i + 1, 'a');
        INSERT INTO test.tbl (pk, val) VALUES (i, 'a');
        SET i = i + 2;
    END WHILE;
END //

DELIMITER ;

-- Call the procedure
CALL PopulateTable();
```

After executing the script, you will have created the table `tbl` with **100,054 records**, each having a size of **262 bytes**. The resulting B+ Tree structure will be highly sparse, as illustrated in the following diagram:

<img src="/public/images/2025-01-14/1.png" alt="image-1" width="80%" />

### Expected B+ Tree Structure

- **Leaf Level**: The table's B+ Tree uses **49,986 leaf pages**, requiring **49,986 × 16KB = 781MB** of space to store only **24MB** of actual records (100,054 × 262 bytes).

## 2. Using ibdNinja to Analyze and Verify the Table

### 1️⃣ **Verifying that the Leftmost Page at Level 0 Contains 29 Records**

First, run the following command to get an overview of the `tbl.ibd` data file:

```
./ibdNinja -f ../innodb-run/mysqld/data/test/tbl.ibd
```

<img src="/public/images/2025-01-14/2.png" alt="image-2" width="80%" />

From the output, you can see that the table contains a B+ Tree index with **Index ID = 547**.

Next, run the following command to get the number of levels in the B+ Tree and the `page ID` of the leftmost page at each level:

```
./ibdNinja -f ../innodb-run/mysqld/data/test/tbl.ibd -e 547
```

<img src="/public/images/2025-01-14/3.png" alt="image-3" width="80%" />

The output will show that the **leftmost page ID at Level 0 (leaf level) is 5**. Let's analyze this page:

```
./ibdNinja -f ../innodb-run/mysqld/data/test/tbl.ibd -p 5
```

The tool will display detailed information about all the records in Page 5. Let's take a look at the last record as an example:

<img src="/public/images/2025-01-14/4.png" alt="image-4" width="80%" />

- **Record Size**: 262 bytes (including a 5-byte header and 257-byte body), which matches our expectation.

Finally, the tool will print a summary of Page 5:

<img src="/public/images/2025-01-14/5.png" alt="image-5" width="80%" />

- **Record Count**: 29 records
- **Space Utilization**: 46.37%

This confirms our expectation that the leftmost page contains 29 records.

### 2️⃣ **Verifying that Subsequent Pages Contain Only 2 Records**

We expect that, starting from the second page, each page in the leaf level will contain only **2 records**, making the tree extremely sparse. Let's verify this.

From the analysis of Page 5, we know that its **next page ID is 7**. 

<img src="/public/images/2025-01-14/6.png" alt="image-6" width="80%" />

Let's analyze Page 7:

```
./ibdNinja -f ../innodb-run/mysqld/data/test/tbl.ibd -p 7
```

<img src="/public/images/2025-01-14/7.png" alt="image-7" width="80%" />

The tool shows that Page 7 indeed contains only **2 records**, with a **space utilization of just 3.20%**, confirming our expectation.

### 3️⃣ **Verifying the Overall Sparsity of the B+ Tree**

To analyze the entire B+ Tree, run the following command:

```
./ibdNinja -f ../innodb-run/mysqld/data/test/tbl.ibd -i 547
```

The tool will print a summary:

<img src="/public/images/2025-01-14/8.png" alt="image-8" width="80%" />

- **Number of Levels**: 3
- **Number of Non-Leaf Pages**: 44
- **Number of Leaf Pages**: 49,986

The tool then provides detailed statistics for both non-leaf and leaf levels:

<img src="/public/images/2025-01-14/9.png" alt="image-9" width="80%" />

- **Non-Leaf Page Utilization**: 90.22% (not sparse)
- **Leaf Page Utilization**: 3.20% (extremely sparse)

This confirms that the B+ Tree is highly sparse at the leaf level.

### 4️⃣ **Making the Situation Even Worse**

The leaf level currently has a utilization rate of only **3.20%**, which is already very low. But we can make it worse by dropping a column from the table.

Since the `val` column (a `CHAR(240)`) occupies **240 bytes** of each record, we can drop this column using **Instant Drop Column**. The space previously occupied by this column remains allocated but becomes unusable, creating unutilizable gaps in the pages

To drop the column, execute the following command in the MySQL client:

```
ALTER TABLE tbl DROP COLUMN val;
```

Then, reanalyze the B+ Tree using the `ibdNinja` tool:

```
./ibdNinja -f ../innodb-run/mysqld/data/test/tbl.ibd -i 547
```

<img src="/public/images/2025-01-14/10.png" alt="image-10" width="80%" />

The analysis shows that **2.93%** of the space in the index is occupied by dropped columns, which cannot be reused. The free space ratio remains unchanged.

### 5️⃣ **Releasing Space with OPTIMIZE TABLE**

To reclaim the wasted space, run the following command in the MySQL client:

```
OPTIMIZE TABLE tbl;
```

After running this command, analyze the B+ Tree again (note that the **Index ID will change**):

```
./ibdNinja -f ../innodb-run/mysqld/data/test/tbl.ibd -i 548
```

<img src="/public/images/2025-01-14/11.png" alt="image-11" width="80%" />

The leaf level's size is reduced from **781MB to 2MB**, with a space utilization of **90%**. The B+ Tree is no longer sparse.



The **ibdNinja** tool provides a practical way to explore `.ibd` files and gain insights into InnoDB's internal structures. It can be helpful for understanding MySQL's storage engine and troubleshooting data-related issues. Whether you're exploring InnoDB for learning or operational purposes, this tool offers assistance.