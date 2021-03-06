# MySQL基本概念－－锁

介绍下对于MySQL锁机制的理解

从基本概念开始：

### **共享锁**

共享锁的代号是S，是Share的缩写，共享锁的锁粒度是行或者元组（多个行）。一个事务获取了共享锁之后，可以对锁定范围内的数据执行读操作。

### **排它锁**

排它锁的代号是X，是`eXclusive`的缩写，排它锁的粒度与共享锁相同，也是行或者元组。一个事务获取了排它锁之后，可以对锁定范围内的数据执行写操作。

假设有两个事务t1和t2

如果事务t1获取了一个元组的共享锁，事务t2还可以立即获取这个元组的共享锁，但不能立即获取这个元组的排它锁（必须等到t1释放共享锁之后）。

如果事务t1获取了一个元组的排它锁，事务t2不能立即获取这个元组的共享锁，也不能立即获取这个元组的排它锁（必须等到t1释放排它锁之后）

### **意向锁**

意向锁是一种表锁，锁定的粒度是整张表，分为意向共享锁(IS)和意向排它锁(IX)两类。意向共享锁表示一个事务有意对数据上共享锁或者排它锁。“有意”这两个字表达的意思比较微妙，说的明白点就是指事务想干这个事但还没真去干。举例说明下意向共享锁，比如一个事务t执行了这样一个语句：`select * from table lock in share model` ，如果这个语句执行成功，就对表table上了一个意向共享锁。**lock in share model**就是说事务t1在接下来要执行的语句中要获取S锁。如果t1的`select * from table lock in share model`执行成功，那么接下来t1应该可以畅通无阻的去执行只需要共享锁的语句了。意向排它锁的含义同理可知，上例中要获取意向排它锁，可以使用`select * from table for update**** `。

**lock in share model 和 for update**这两个东西在数据率理论中还有个学名叫悲观锁，与悲观锁相对的当然还有乐观锁。大家可以看到各种锁都是成双成对出现的。关于悲观锁和乐观锁的问题暂且不表，下文再来详述。

### **锁的互斥与兼容关系**

锁和锁之间的关系，要么是相容的，要么是互斥的。

锁a和锁b相容是指：操作同样一组数据时，如果事务t1获取了锁a,另一个事务t2还可以获取锁b；

锁a和锁b互斥是指：操作同样一组数据时，如果事务t1获取了锁a，另一个事务t2在t1释放锁a之前无法获取锁b。

上面提到的共享锁、排它锁、意向共享锁、意向排它锁相互之前都是有兼容/互斥关系的，可以用一个兼容性矩阵表示(y表示兼容，n表示不兼容):

> ```java
>    X    S    IX    IS
> X  n     n    n     n
> S  n     y    n     y
> IX n     n    y     y
> IS n     y    y     y  
> ```

兼容性矩阵为什么是这个样子的？

X和S的相互关系在上文中解释过了，IX和IS的相互关系全部是兼容，这也很好理解，因为它们都只是“有意”，还处于YY阶段，没有真干，所以是可以兼容的；

剩下的就是X和IX，X和IS, S和IX， S和IS的关系了，我们可以由X和S的关系推导出这四组关系。

简单的说：X和IX的=X和X的关系。为什么呢？因为事务在获取IX锁后，接下来就有权利获取X锁。如果X和IX兼容的话，就会出现两个事务都获取了X锁的情况，这与我们已知的X与X互斥是矛盾的，所以X与IX只能是互斥关系。其余的三组关系同理，可用同样的方式推导出来。

 

**对于 UPDATE、DELETE和INSERT语句，InnoDB会自动给涉及数据集加排他锁（X)；对于普通SELECT语句，InnoDB不会加任何锁； 事务可以通过显式给记录集加共享锁或排他锁。**

### **其它：**

#### 一致性非阻塞读

select... lock in share mode和select ... for update的区别

索引记录锁

间隙锁

后码锁

各种语句对应的锁类型

在有索引的情况下是以后码锁为基础的行级锁，在固定索引键查找的情况下是索引记录锁，在没有可用索引的情况下上升到表锁

有索引的情况：

select ... from 一致性非阻塞读，不上锁。在serializable隔离级别下例外，在这个隔离级别下上共享后码锁

select ... from ... lock in share mode  共享后码锁

select ... from ... for update 排它后码锁

update .... where  排它后码锁

delete from .... where 排它后码锁

insert ... 排它索引记录锁，如果发生键值唯一性冲突则转成共享锁

insert ... on duplicate key update ，一直都是排它锁

replace ... 一直都是排它锁