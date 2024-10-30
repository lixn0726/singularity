# B-Trees and database indexes

Date: 2024.10.22

Link: https://planetscale.com/blog/btrees-and-database-indexes

## Comparing different primary key choices

1. auto-increment
2. uuid



### UUID

- inserting

  1. unpredicatable inserting path
  2. unpredicatable inserting destination nodes
  3. values stored in leaves are not in order

  1, 2 lead to a poor performance issue, because needs to get different pages(disk blocks) into cache each time inserting.

  3 leads to poor performance issue when searching data.

- reading

  1. range reading, the destination nodes are separated in different place

- size

  



### Auto-increment integer

- inserting

  1. always follow the right-most path
  2. leaves only added on the right side
  3. leaves value always in sorted order

  1, 2 lead to better performance, for the higher cache hit rate.

- reading
  1. range reading, the destination nodes are all in an area, means they are near to each other 
- size



## B+trees, Pages and InnoDB

Big benefits of a B+tree is the fact taht we can set the node size to whatever we want.

InnoDB, the **B+tree nodes** are set to 16k, which is same to the **InnoDB page**.





## Tips

Split percentage, affecting the nodes visited when inserting data.

Default 50% is a good choice, reference to the original link, the chapter: [Primary key choice: insertions](https://planetscale.com/blog/btrees-and-database-indexes#primary-key-choice-insertions)



We want the PK to be:

1. Big enough to never face exhaustion
2. Small enough to not use excessive storage

UUID: 16 bytes

auto-inc: from int(4 bytes), to bigint(8 bytes)

Comapring BIGINT and UUID, within the same size of nodes, it can fit in more BIGINT PK.

so, BIGINT(auto-inc) PK is better.







## Conclusion

1. auto-increment integer is better choice for PK comparing to UUID
2. 