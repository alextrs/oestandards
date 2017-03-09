# OpenEdge Standards

## Table of Contents
1. [Objects](#objects)
1. [Error Handling](#error-handling)
1. [Data Access](#data-access)
1. [Comments](#comments)
1. [Performance](#performance)
1. [Variables](#variables)

## Objects

## Error Handling
<a name="no--error"></a><a name="1.1"></a>
  - [2.1](#no--error) **NO-ERROR**: Use NO-ERROR only when you expect an error to occur and if used handle error appropriately
    > Why? NO-ERROR suppresses errors, which can cause database consistency, memory leaks, infinite loops and more...

    ```openedge
    /* bad (error is suppressed, cMemberName is never assigned */
    ASSIGN iMemberNumber = INTEGER("ABC")
           cMemberName   = 'ABC' NO-ERROR.

    /* good (ver 1) - using structured error handling */
    ASSIGN iMemberNumber = INTEGER("ABC")
           cMemberName   = 'ABC'.

    /* ... some code... */

    CATCH eExc AS Progress.Lang.ProError:
      MESSAGE "Error:" + eExc:GetMessage(1).
    END.

    /* good (ver 1) */
    ASSIGN iMemberNumber = INTEGER("ABC")
           cMemberName   = 'ABC' NO-ERROR.
    IF ERROR-STATUS:ERROR THEN
      DO:
        /* handle error here */
      END.

    /* ... some code... */

    CATCH eExc AS Progress.Lang.ProError:
      /* handle error here */
    END.

    ```

## Data Access

<a name="record--locking"></a><a name="1.1"></a>
  - [3.1](#record--locking) **Record Locking**: Always use either NO-LOCK or EXCLUSIVE-LOCK
    > Why? If you don't specify locking mechanism SHARE-LOCK is used. NO-LOCK has better performance over SHARE-LOCK. Other users can't obtain EXCLUSIVE-LOCK on record that SHARE locked

    ```openedge
    /* bad */
    FIND FIRST member
         WHERE member.id EQ 0.346544767...
    FOR EACH member:...
    CAN-FIND (FIRST member WHERE member.id EQ 0.346544767)...

    /* good */
    FIND FIRST member NO-LOCK
         WHERE member.id EQ 0.346544767...
    FOR EACH member NO-LOCK:...
    CAN-FIND (FIRST member NO-LOCK
              WHERE member.id EQ 0.346544767)...
    ```

  - [1.1](#record--locking) **Record Locking**: Always use either NO-LOCK or EXCLUSIVE-LOCK
    > Why? If you don't specify locking mechanism SHARE-LOCK is used. NO-LOCK has better performance over SHARE-LOCK. Other users can't obtain EXCLUSIVE-LOCK on record that SHARE locked

    ```openedge
    ```
## Comments

## Performance

## Variables
