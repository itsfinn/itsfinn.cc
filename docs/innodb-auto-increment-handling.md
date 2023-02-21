---
title: "InnoDB 如何处理 AUTO_INCREMENT"
date: 2023-02-12T12:00:00+08:00
isCJKLanguage: true
Description: "kubernetes 资源名称规范及其正则表达式"
Tags: ["MySQL", "InnoDB", "AUTO_INCREMENT"]
Categories: ["MySQL"]
---


# InnoDB 中的 AUTO_INCREMENT 处理


InnoDB 提供了一种可配置的锁定机制，可以显着提高将行添加到具有 AUTO_INCREMENT 列的表的 SQL 语句的可伸缩性和性能。要对 InnoDB 表使用 AUTO_INCREMENT 机制，必须将 AUTO_INCREMENT 列定义为某个索引的第一列或唯一列，这样就可以在表上执行与索引 SELECT MAX(ai_col) 查找等效的操作以获得最大列值。索引不需要是 PRIMARY KEY 或 UNIQUE ，但为了避免 AUTO_INCREMENT 列中的重复值，建议使用这些索引类型。

本节介绍 AUTO_INCREMENT 锁定模式、不同 AUTO_INCREMENT 锁定模式设置的使用含义以及 InnoDB 如何初始化 AUTO_INCREMENT 计数器。


## InnoDB AUTO_INCREMENT 锁定模式


本节介绍用于生成自动增量值的 AUTO_INCREMENT 锁定模式，以及每种锁定模式如何影响复制。自增锁定模式在启动时使用 `innodb_autoinc_lock_mode` 变量进行配置。

以下术语用于描述 `innodb_autoinc_lock_mode` 设置：


- “INSERT-like”语句

    在表中生成新行的所有语句，包括 INSERT 、 INSERT ... SELECT 、 REPLACE 、 REPLACE ... SELECT 和 LOAD DATA 。包括“简单插入”、“批量插入”和“混合模式”插入。

- “简单插入”

    可以预先确定要插入的行数的语句（当语句最初被处理时）。这包括没有嵌套子查询的单行和多行 INSERT 和 REPLACE 语句，但不包括 INSERT ... ON DUPLICATE KEY UPDATE 。

- “批量插入”

    事先不知道要插入的行数（以及所需的自动增量值的数量）的语句。这包括 INSERT ... SELECT 、 REPLACE ... SELECT 和 LOAD DATA 语句，但不包括普通的 INSERT 。在处理每一行时， InnoDB 一次为 AUTO_INCREMENT 列分配一个新值。

- “混合模式插入”

    这些是“简单插入”语句，它们为一些（但不是全部）新行指定自动增量值。下面是一个示例，其中 c1 是表 t1 的 AUTO_INCREMENT 列：

    ```sql
    INSERT INTO t1 (c1,c2) VALUES (1,'a'), (NULL,'b'), (5,'c'), (NULL,'d');
    ```

    另一种类型的“混合模式插入”是 INSERT ... ON DUPLICATE KEY UPDATE ，在最坏的情况下，它实际上是一个 INSERT 后跟一个 UPDATE ，其中为 AUTO_INCREMENT 列分配的值可能会也可能不会被使用更新阶段。


`innodb_autoinc_lock_mode` 变量有三种可能的设置。 “传统”、“连续”或“交错”锁定模式的设置分别为 0、1 或 2。

- innodb_autoinc_lock_mode = 0 （“传统”锁定模式）

    传统的锁定模式提供了与引入 innodb_autoinc_lock_mode 变量之前相同的行为。由于语义上可能存在差异，提供传统锁定模式选项是为了向后兼容、性能测试和解决“混合模式插入”的问题。

    在这种锁定模式下，所有“INSERT-like”的语句都获得一个特殊的表级 AUTO-INC 锁，用于插入到具有 AUTO_INCREMENT 列的表中。这个锁通常保持到语句的末尾（而不是事务的末尾），这样就可以保证, 当给定的多条 INSERT 语句顺序确定时, 给每条语句分配的值(自增列), 是可预测可重复的, 并且是连续的。

    对于基于语句的复制，这意味着当在副本服务器上复制 SQL 语句时，自动增量列使用与源服务器上相同的值。多个 INSERT 语句的执行结果是确定性的，副本重现与源相同的数据。如果由多个 INSERT 语句生成的自动增量值交错，则两个并发 INSERT 语句的结果将是不确定的，并且无法可靠地传播到使用基于语句的复制的副本服务器。

    为清楚起见，请考虑使用此表的示例：

    ```sql
    CREATE TABLE t1 (
        c1 INT(11) NOT NULL AUTO_INCREMENT,
        c2 VARCHAR(10) DEFAULT NULL,
        PRIMARY KEY (c1)
    ) ENGINE=InnoDB;
    ```

    假设有两个事务正在运行，每个事务都将行插入到具有 AUTO_INCREMENT 列的表中。一个事务使用插入 1000 行的 INSERT ... SELECT 语句，另一个事务使用插入一行的简单 INSERT 语句：

    > Tx1: INSERT INTO t1 (c2) SELECT 1000 rows from another table ...
    >
    > Tx2: INSERT INTO t1 (c2) VALUES ('xxx');

    InnoDB 无法预先知道在 Tx1 中的 INSERT 语句中从 SELECT 中检索了多少行，它会随着语句的进行一次分配一个自动增量值。使用表级锁，一直保持到语句末尾，一次只能执行一条引用 t1 表的 INSERT 语句，不同语句生成自增数不会交错。 Tx1 INSERT ... SELECT 语句生成的自增值是连续的，而 Tx2 中 INSERT 语句使用的（单个）自增值小于或大于所有用于 Tx1 的自增值，具体取决于哪个语句先执行。

    只要 SQL 语句在从二进制日志重播时（使用基于语句的复制时，或在恢复场景中）以相同的顺序执行，结果就与 Tx1 和 Tx2 首次运行时相同。因此，表级锁一直保持到语句结束，使 INSERT 语句使用自动增量安全地用于基于语句的复制。但是，当多个事务同时执行插入语句时，那些表级锁会限制并发性和可伸缩性。

    在前面的例子中，如果没有表级锁，那么Tx2中 INSERT 使用的自增列的值恰好取决于语句执行的时间。如果 Tx2 的 INSERT 在 Tx1 的 INSERT 运行时执行（而不是在它开始之前或完成之后），则两个 INSERT 语句分配的具体自动增量值是不确定的，并且可能与变来变去。

    在连续锁模式下， InnoDB 可以避免对行数预先已知的“简单插入”语句使用表级锁 AUTO-INC ，并且仍然保留确定的执行结果和基于语句的复制的安全性。

    如果您不使用二进制日志来重放 SQL 语句作为恢复或复制的一部分，则可以使用交错锁模式来消除对表级 AUTO-INC 锁的所有使用，以获得更高的并发性和性能，但代价是允许语句分配的自动递增数字中的间隙，并可能交错并发执行的语句分配的数字。

- innodb_autoinc_lock_mode = 1 （“连续”锁定模式）

    这是默认的锁定模式。在这种模式下，“批量插入”使用特殊的 AUTO-INC 表级锁并将其保持到语句结束。这适用于所有 INSERT ... SELECT 、 REPLACE ... SELECT 和 LOAD DATA 语句。一次只能执行一条持有 AUTO-INC 锁的语句。如果批量插入操作的源表与目标表不同，则在对从源表中选择的第一行加共享锁后，再加对目标表的 AUTO-INC 锁。如果批量插入操作的源和目标是同一张表，那么 AUTO-INC 锁是在对所有选中的行加共享锁后加的。

    “简单插入”（预先知道要插入的行数）通过在互斥锁（mutex,轻量级锁,仅在分配自动增量值过程期间保留，不需要保留到语句完成）的控制下获取所需数量的自动增量值来避免表级 AUTO-INC 锁。除非另一个事务持有 AUTO-INC 锁，否则不使用表级 AUTO-INC 锁。如果另一个事务持有 AUTO-INC 锁，则“简单插入”会等待 AUTO-INC 锁，就好像它是“批量插入”一样。

    这种锁定模式确保，在存在行数事先未知的 INSERT 语句（所需的自动递增数字,随着语句的进行分配）的情况下，所有由任何“INSERT-like”语句分配的自动递增值是连续的，对于基于语句的复制是安全的。

    简而言之，这种锁定模式显着提高了可伸缩性，同时可以安全地用于基于语句的复制。此外，与“传统”锁定模式一样，任何给定语句分配的自动递增数字都是连续的。对于任何使用自动递增的语句，与“传统”模式相比，语义没有变化，但有一个重要的例外。

    例外情况是“混合模式插入”，其中用户为多行“简单插入”中的一些（但不是全部）行的 AUTO_INCREMENT 列提供显式值。对于此类插入， InnoDB 分配的自动增量值多于要插入的行数。但是，所有自动分配的值都是连续生成的（因此高于）由最近执行的先前语句生成的自动增量值。 “多余”的数字丢失了。

- innodb_autoinc_lock_mode = 2 （“交错”锁定模式）

    在这种锁模式下，所有“INSERT-like”语句都不会使用表级 AUTO-INC 锁，多个语句可以同时执行。这是最快和最具扩展性的锁定模式，但在使用基于语句的复制或从二进制日志重放 SQL 语句的恢复场景时，它并不安全。

    在这种锁定模式下，InnoDB保证在所有并发执行的“INSERT-like” 语句中, 自增值是唯一且单调递增的。但是，由于多个语句可以同时生成数字（即，数字的分配在语句之间交错），因此一条插入多行记录的语句, 分配到值可能是不连续的。

    如果只执行一条“简单插入”语句，其中要插入的行数提前已知，则为单个语句生成的数字没有间隙，“混合模式插入”除外。但是，当执行“批量插入”时，任何给定语句分配的自动增量值可能存在间隙。

## InnoDB AUTO_INCREMENT 锁模式使用含义

- 在复制中使用自动增量

    如果您使用的是基于语句的复制，请将 `innodb_autoinc_lock_mode` 设置为 0 或 1 并在源及其副本服务器上使用相同的值。如果您使用 `innodb_autoinc_lock_mode = 2`（“交错”）或源和副本不使用相同锁定模式的配置，则不能确保副本上的自动增量值与源上的值相同。

    如果您使用基于行或混合格式的复制，所有的自增锁模式都是安全的，因为基于行的复制对 SQL 语句的执行顺序不敏感（并且对于混合格式的复制, 如果一条语句基于语句的复制是不安全的, 则会使用使用基于行的复制）。

- “丢失”的自动增量值和序列间隙

    在所有锁定模式（0、1 和 2）中，如果生成自动增量值的事务回滚，则这些自动增量值将“丢失”。自增列一旦产生值，就不能回滚，不管“INSERT-like”语句是否完成，也不管事务是否被执行回滚。这些丢失的值不会被重用。因此，存储在表的 AUTO_INCREMENT 列中的值可能存在间隙。

- 为 AUTO_INCREMENT 列指定 NULL 或 0

    在所有锁定模式（0、1 和 2）中，如果用户为 INSERT 中的 AUTO_INCREMENT 列指定 NULL 或 0，则 InnoDB 将该行视为未指定值, 并为它生成一个新值。

- 将负值分配给 AUTO_INCREMENT 列

    在所有锁定模式（0、1 和 2）中，如果将负值分配给 AUTO_INCREMENT 列，则自动递增机制的行为是未定义的。

- 如果 AUTO_INCREMENT 值变得大于指定整数类型的最大整数

    在所有锁定模式（0、1 和 2）中，如果值变得大于可以存储在指定整数类型中的最大整数，则自动递增机制的行为是未定义的。

- “批量插入”的自动增量值差距

    `innodb_autoinc_lock_mode` 设置为 0（“传统”）或 1（“连续”）时，任何给定语句生成的自动增量值都是连续的，没有间隙，因为表级 AUTO-INC 锁一直保持到结束的语句，并且一次只能执行一个这样的语句。

    `innodb_autoinc_lock_mode` 设置为 2（“interleaved”）时，“批量插入”生成的自动增量值可能存在间隙，但前提是同时执行其他“INSERT-like”语句。

    对于锁定模式 1 或 2，连续语句之间可能会出现间隙，因为对于批量插入，可能不知道每个语句所需的自动增量值的确切数量，并且可能会高估。

- 由“混合模式插入”分配的自动增量值

    考虑“混合模式插入”，其中“简单插入”指定某些（但不是全部）结果行的自动增量值。这样的语句在锁定模式 0、1 和 2 中的行为不同。例如，假设 c1 是表 t1 的 AUTO_INCREMENT 列，并且最近自动生成的序列号是 100。

    ```shell
    mysql> CREATE TABLE t1 (
        ->  c1 INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY, 
        ->  c2 CHAR(1)
        -> ) ENGINE = INNODB;
    ```

    现在，考虑以下“混合模式插入”语句：
    
    ```shell
    mysql> INSERT INTO t1 (c1,c2) VALUES (1,'a'), (NULL,'b'), (5,'c'), (NULL,'d');
    ```
    
    `innodb_autoinc_lock_mode` 设置为 0（“传统”），四个新行是：
    
    ```shell
    mysql> SELECT c1, c2 FROM t1 ORDER BY c2;
    +-----+------+
    | c1  | c2   |
    +-----+------+
    |   1 | a    |
    | 101 | b    |
    |   5 | c    |
    | 102 | d    |
    +-----+------+
    ```
    下一个可用的自动增量值是 103，因为自动增量值是一次分配一个，而不是在语句执行开始时一次分配。无论是否同时执行“INSERT-like”语句（任何类型），此结果都是正确的。

    `innodb_autoinc_lock_mode` 设置为 1（“连续”），四个新行也是：

    ```shell
    mysql> SELECT c1, c2 FROM t1 ORDER BY c2;
    +-----+------+
    | c1  | c2   |
    +-----+------+
    |   1 | a    |
    | 101 | b    |
    |   5 | c    |
    | 102 | d    |
    +-----+------+
    ```

    但是，在这种情况下，下一个可用的自动增量值是 105，而不是 103，因为在处理语句时分配了四个自动增量值，但只使用了两个。无论是否同时执行“INSERT-like”语句（任何类型），此结果都是正确的。

    `innodb_autoinc_lock_mode` 设置为 2（“交错”），四个新行是：

    ```shell
    mysql> SELECT c1, c2 FROM t1 ORDER BY c2;
    +-----+------+
    | c1  | c2   |
    +-----+------+
    |   1 | a    |
    |   x | b    |
    |   5 | c    |
    |   y | d    |
    +-----+------+
    ```

    x 和 y 的值是唯一的，并且比之前生成的任何行都大。但是 x 和 y 的具体取值取决于并发执行语句产生的自增值的个数。

    最后，考虑以下语句，在最近生成的序列号为 100 时发出：

    ```shell
    mysql> INSERT INTO t1 (c1,c2) VALUES (1,'a'), (NULL,'b'), (101,'c'), (NULL,'d');
    ```
    
    对于任何 innodb_autoinc_lock_mode 设置，此语句都会生成重复键错误 23000 ( Can't write; duplicate key in table )，因为为行 (NULL, 'b') 分配了 101，而行 (101, 'c') 的插入失败。

- 修改 INSERT 语句序列中间的 AUTO_INCREMENT 列值
    
    在所有锁定模式（0、1 和 2）中，修改 INSERT 语句序列中间的 AUTO_INCREMENT 列值可能会导致“重复输入”错误。例如，如果执行 UPDATE 操作将 AUTO_INCREMENT 列值更改为大于当前最大自动增量值的值，则后续未指定未使用的自动增量值的 INSERT 操作可能会遇到“重复输入”错误。以下示例演示了此行为。

    ```shell
    mysql> CREATE TABLE t1 (
        -> c1 INT NOT NULL AUTO_INCREMENT,
        -> PRIMARY KEY (c1)
        ->  ) ENGINE = InnoDB;

    mysql> INSERT INTO t1 VALUES(0), (0), (3);

    mysql> SELECT c1 FROM t1;
    +----+
    | c1 |
    +----+
    |  1 |
    |  2 |
    |  3 |
    +----+

    mysql> UPDATE t1 SET c1 = 4 WHERE c1 = 1;

    mysql> SELECT c1 FROM t1;
    +----+
    | c1 |
    +----+
    |  2 |
    |  3 |
    |  4 |
    +----+

    mysql> INSERT INTO t1 VALUES(0);
    ERROR 1062 (23000): Duplicate entry '4' for key 'PRIMARY'
    ```

## InnoDB AUTO_INCREMENT 计数器初始化

本节介绍 InnoDB 如何初始化 AUTO_INCREMENT 计数器。

如果为 InnoDB 表指定 AUTO_INCREMENT 列，则 InnoDB 数据字典中的表句柄包含一个称为自动递增计数器的特殊计数器，用于为该列分配新值。该计数器仅存储在主内存中，而不存储在磁盘上。

为了在服务器重新启动后初始化一个自动递增计数器， InnoDB 在第一次插入包含 AUTO_INCREMENT 列的表时执行与以下语句等效的语句。
```sql
SELECT MAX(ai_col) FROM table_name FOR UPDATE;
```
InnoDB 递增语句检索的值并将其分配给列和表的自动递增计数器。默认情况下，该值递增 1。可以通过 `auto_increment_increment` 配置设置覆盖此默认值。

如果表为空， InnoDB 使用值 1 。这个默认值可以被 `auto_increment_offset` 配置设置覆盖。

如果 SHOW TABLE STATUS 语句在自动递增计数器初始化之前检查表，则 InnoDB 初始化但不递增该值。存储该值供以后插入使用。此初始化在表上使用正常的独占锁定读取，锁定持续到事务结束。 InnoDB 遵循相同的过程为新创建的表初始化自动递增计数器。

自动递增计数器初始化后，如果您没有明确指定 AUTO_INCREMENT 列的值， InnoDB 会递增计数器并将新值分配给该列。如果插入一行明确指定列值，并且该值大于当前计数器值，则计数器设置为指定的列值。

只要服务器运行， InnoDB 就会使用内存中的自动递增计数器。当服务器停止并重新启动时， InnoDB 为第一个 INSERT 重新初始化每个表的计数器，如前所述。

服务器重新启动还会取消 CREATE TABLE 和 ALTER TABLE 语句中 AUTO_INCREMENT = N 表选项的影响，您可以将其与 InnoDB 表一起使用以设置初始计数器值或更改当前计数器值。

## 注释

当 AUTO_INCREMENT 整数列用完值时，后续的 INSERT 操作将返回重复键错误。这是一般的 MySQL 行为。

当您重新启动 MySQL 服务器时， InnoDB 可能会重用为 AUTO_INCREMENT 列生成但从未存储的旧值（即，在回滚的旧事务期间生成的值）。