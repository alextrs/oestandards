# OpenEdge Standards

## Table of Contents
1. [Objects](#objects)
1. [Data Access](#data-access)
1. [Comments](#comments)
1. [Performance](#performance)
1. [Variables](#variables)

## Objects

## Data Access

<a name="record--locking"></a><a name="1.1"></a>
  - [1.1](#record--locking) **Record Locking**: Always use either NO-LOCK or EXCLUSIVE-LOCK
    > Why? If you don't specify locking mechanism SHARE-LOCK is used.
      - NO-LOCK has better performance over SHARE-LOCK.
      - Other users can't obtain EXCLUSIVE-LOCK on record that SHARE locked

    ```openedge
    // bad
    FIND FIRST member
         WHERE member.id EQ 0.346544767...
    FOR EACH member:...
    CAN-FIND (FIRST member WHERE member.id EQ 0.346544767)...

    // good
    FIND FIRST member NO-LOCK
         WHERE member.id EQ 0.346544767...
    FOR EACH member NO-LOCK:...
    CAN-FIND (FIRST member NO-LOCK
              WHERE member.id EQ 0.346544767)...
    ```

## Comments

## Performance

## Variables
