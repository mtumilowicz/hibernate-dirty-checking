[![Build Status](https://travis-ci.com/mtumilowicz/hibernate-dirty-checking.svg?token=PwyvjePQ7aiAX51hSYLE&branch=master)](https://travis-ci.com/mtumilowicz/hibernate-dirty-checking)

# hibernate-dirty-checking
Hibernate dirty-checking mechanism description.

_Reference_: http://docs.jboss.org/hibernate/orm/5.3/userguide/html_single/Hibernate_User_Guide.html  
_Reference_: https://vladmihalcea.com/the-anatomy-of-hibernate-dirty-checking/  
_Reference_: https://vladmihalcea.com/how-to-customize-hibernate-dirty-checking-mechanism/

# preface
Historically Hibernate only supported diff-based dirty calculation for 
determining which entities in a persistence context have changed. This 
essentially means that Hibernate would keep track of the last known 
state of an entity in regards to the database (typically the last read 
or write). Then, as part of flushing the persistence context, Hibernate 
would walk every entity associated with the persistence context and check 
its current state against that "last known database state". This is by 
far the most thorough approach to dirty checking because it accounts for 
data-types that can change their internal state (java.util.Date is the 
prime example of this). However, in a persistence context with a large 
number of associated entities, it can also be a performance-inhibiting 
approach.

If your application does not need to care about "internal state changing 
data-type" use cases, bytecode-enhanced dirty tracking might be a 
worthwhile alternative to consider, especially in terms of performance. 
In this approach Hibernate will manipulate the bytecode of your classes 
to add "dirty tracking" directly to the entity, allowing the entity itself 
to keep track of which of its attributes have changed. During the flush 
time, Hibernate asks your entity what has changed rather than having to 
perform the state-diff calculations.

# internals
The **UPDATE** statement is generated by EntityUpdateAction during flushing 
if the managed entity has been marked modified. The dirty checking 
mechanism is responsible for determining if a managed entity has been 
modified since it was first loaded.

By default Hibernate checks all managed entity properties. Every time an 
entity is loaded, Hibernate makes an additional copy of all entity property 
values. At flush time, every managed entity property is matched against 
the loading-time snapshot value.

For a large number of managed entities, the default dirty checking 
mechanism may have a significant CPU and memory footprint. Since the 
initial entity snapshot is held separately, the persistence context 
requires twice as much memory as all managed entities would normally 
occupy.

A more efficient approach would be to mark dirty properties upon 
value changing. The bytecode enhancement (weaving types) can 
happen at:
* **Build-time** (Maven, Gradle will modify code just after compilation)
* **Runtime** (during class loading)

# immutability
Important aspect of dirty-checking is its connection with `@Immutable`
entities. In short: immutable entities are not tracked by this mechanism.

To get more information please refer my other project
https://github.com/mtumilowicz/hibernate-immutable

# project description
We log all queries (`application.properties`):
```
spring.jpa.properties.hibernate.show_sql=true
```
and in `DirtyCheckTest` we perform operations to show how dirty-checking
in hibernate works:
* `noChange`
    ```
    noChange start
    Hibernate: select customer0_.id as id1_0_0_, customer0_.name as name2_0_0_ from customer customer0_ where customer0_.id=?
    Hibernate: select customer0_.id as id1_0_1_, customer0_.name as name2_0_1_, issues1_.issues_id as issues_i3_1_3_, issues1_.id as id1_1_3_, issues1_.id as id1_1_0_, issues1_.description as descript2_1_0_ from customer customer0_ left outer join issue issues1_ on customer0_.id=issues1_.issues_id where customer0_.id=?
    noChange end
    ```
* `field_change`
    ```
    field_change start
    Hibernate: select customer0_.id as id1_0_0_, customer0_.name as name2_0_0_ from customer customer0_ where customer0_.id=?
    Hibernate: select customer0_.id as id1_0_1_, customer0_.name as name2_0_1_, issues1_.issues_id as issues_i3_1_3_, issues1_.id as id1_1_3_, issues1_.id as id1_1_0_, issues1_.description as descript2_1_0_ from customer customer0_ left outer join issue issues1_ on customer0_.id=issues1_.issues_id where customer0_.id=?
    Hibernate: update customer set name=? where id=?
    field_change end    
    ```
* `change_in_collection`
    ```
    change_in_collection start
    Hibernate: select customer0_.id as id1_0_0_, customer0_.name as name2_0_0_, issues1_.issues_id as issues_i3_1_1_, issues1_.id as id1_1_1_, issues1_.id as id1_1_2_, issues1_.description as descript2_1_2_ from customer customer0_ left outer join issue issues1_ on customer0_.id=issues1_.issues_id where customer0_.id=?
    Hibernate: select customer0_.id as id1_0_1_, customer0_.name as name2_0_1_, issues1_.issues_id as issues_i3_1_3_, issues1_.id as id1_1_3_, issues1_.id as id1_1_0_, issues1_.description as descript2_1_0_ from customer customer0_ left outer join issue issues1_ on customer0_.id=issues1_.issues_id where customer0_.id=?
    Hibernate: select issue0_.id as id1_1_0_, issue0_.description as descript2_1_0_ from issue issue0_ where issue0_.id=?
    Hibernate: insert into issue (description, id) values (?, ?)
    Hibernate: update issue set issues_id=? where id=?
    change_in_collection end    
    ```
* `field_change_in_relation`
    ```
    field_change_in_relation start
    Hibernate: select customer0_.id as id1_0_0_, customer0_.name as name2_0_0_, issues1_.issues_id as issues_i3_1_1_, issues1_.id as id1_1_1_, issues1_.id as id1_1_2_, issues1_.description as descript2_1_2_ from customer customer0_ left outer join issue issues1_ on customer0_.id=issues1_.issues_id where customer0_.id=?
    Hibernate: select customer0_.id as id1_0_1_, customer0_.name as name2_0_1_, issues1_.issues_id as issues_i3_1_3_, issues1_.id as id1_1_3_, issues1_.id as id1_1_0_, issues1_.description as descript2_1_0_ from customer customer0_ left outer join issue issues1_ on customer0_.id=issues1_.issues_id where customer0_.id=?
    Hibernate: update issue set description=? where id=?
    field_change_in_relation end    
    ```