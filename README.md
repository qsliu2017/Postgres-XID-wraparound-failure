# Postgres-XID-wraparound-failure

PostgresSQL database provides a non-block read/write model by multiversion concurrency control (MVCC).
The MVCC can be simply described as "rows created *in the future* or deleted *in the past* are invisible to current transcation".
To determine whether a row is *in the future* or *in the past* of a trascation, Postgres assigns an incrementing 32-bits interger, **transcation ID** (XID), to each trancation.
For instance, a row created by a transcation has a `XMIN` value equals to XID of the transcation, indicating this row is visible only to transcations in the future of `XMIN`.

## Linear XID space

Given a transcation with `XID` and a row with `XMIN`, there is a basic algorithm to determine the visibility:
```C
bool isVisible(unsigned int XID, unsigned int XMIN){
  return XID > XMIN;
}
```

An obvious problem is that the number of transcations might be beyond the scope of 32-bits interger.
Once the XID increases up to the limit of 32-bits, it wraps around to zero. Then all the rows actually in the past appear to be in the future.
This is so-called **wrap-around** failure.

## Vacuum

To avoid wrap-around failure, we can ***freeze*** the rows which are *old enough*.
Let's say there is a row with `XMIN`, and all the transcations begin in the past are completed.
Then `XMIN` field is unnecessary since any active transcation is in the future of it.
So we can mark this row as *frozen* and reuse the XID. The process of freezing rows and reusing XID is called `VACUUM`.

The word 'vacuum' is quite interesting. Consider XID counter is going through the XIDs space and leaving some used XIDs along the way, while a vacuum cleaner is following and cleaning the space.
![](vacuum.svg)

With the baseline algorithm, XID counter returns to the beginning of XID space once the wrap-around occurs.
However it must be blocked until all XIDs are vacuumed. This could become a bottleneck of database.

## Modulo XID space
In real world, Postgres actually uses a modulo 2^32 arithmetic XID space, where the visibility algorithm can be briefly described as:
```C
bool isVisible(unsigned int XID, unsigned int XMIN){
  return  XID - XMIN < 2^31;
}
```

We can think of XIDs space as a circle. Given a XID, there are XIDs in a half of the space greater than it.

With this XID space, Postgres just need to promise that, whenever a transcation begins, the greater half of the space is vacuumed.
![](modulo-vacuum.svg)

Consider if we do vacuum so aggressive that the XID space is always almost clean up, the XID counter will never be blocked!

## `auto_vacuum`

Postgres provides an `auto_vacuum` deamon to do vacuum. By setting the params of auto_vacuum, database managers can control the action depends on the specific domain.
However unsuitable params might also cause ...