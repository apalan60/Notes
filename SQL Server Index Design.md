# SQL Server Index Design Guide

> [Reference](https://github.com/apalan60/Notes.git)

## [Index basic design](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide?view=sql-server-ver15#index-design-basics)

- Index 用於加速查找到data row, 其本身也會被放在page, 為**index pages**

```txt
[Index Page 1] [Index Page 2] [Index Page 3]
     /  \           /  \           /  \
[Leaf] [Leaf]   [Leaf] [Leaf]   [Leaf] [Leaf]
```

- **Leaf pages** 包含指向Disk實際記憶體區塊的pointers

```txt
 [Index Page 1] [Index Page 2] [Index Page 3]
          |      |       |
  -----------------------------
  |     |     |     |     | ...
[Leaf Page] (each contains many pointers)
       |       |       |  
   [Disk Data Block]  [Disk Data Block]  [Disk Data Block]
```

- 但過多的index page也會使查找緩慢，就像一本有10000頁的書，裡面有1000頁的索引，想找某個關鍵字在哪幾頁，就得先去那1000頁中找出這個關鍵字的索引
  所以有了**Root Page**，在index pages之前，新增一個紀錄index A --> Z分別在index page的哪些範圍
  e.g. A --> D 在index page 121

```txt
             [Root Page]
                 |
          ----------------
          |      |       |
 [Index Page 1] [Index Page 2] [Index Page 3]
          |      |       |
  -----------------------------
  |     |     |     |     | ...
[Leaf Page] (each contains many pointers)
       |       |       |  
   [Disk Data Block]  [Disk Data Block]  [Disk Data Block]

```
