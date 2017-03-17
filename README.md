# OpenEdge Standards

## Table of Contents
1. [Objects](#objects)
1. [Error Handling](#error-handling)
1. [Data Access](#data-access)
1. [Comments](#comments)
1. [Performance](#performance)
1. [Variables](#variables)
1. [Naming Conventions](#naming-conventions)

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

    /* good (ver 2) - classic error handling */
    ASSIGN iMemberNumber = INTEGER("ABC")
           cMemberName   = 'ABC' NO-ERROR.
    IF ERROR-STATUS:ERROR THEN
      DO:
        /* handle error here */
      END.
    ```

## Data Access

<a name="record--locking"></a><a name="3.1"></a>
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
<a name="exp-trans-scope"></a><a name="3.2"></a>
  - [3.2](#exp--trans--scope) **Transaction Scoope**: Always explicitly define the transaction scope

    ```openedge
    /* bad */
    FIND FIRST provider EXCLUSIVE-LOCK NO-ERROR.
    IF AVAILABLE provider THEN
      ASSIGN provider.name = 'New Provider':U.
    /* ... some code... */
    FOR EACH member EXCLUSIVE-LOCK:
      ASSIGN member.memberName = 'New member name':U.
    END.

    /* good (provider should be updated separately from members) */
    DO TRANSACTION:
      FIND FIRST provider EXCLUSIVE-LOCK
           WHERE provider.id EQ 0.657532547 NO-ERROR.
      IF AVAILABLE provider THEN
        ASSIGN provider.name = 'New Provider':U.
    END.
    /* ... some code... */
    FOR EACH member EXCLUSIVE-LOCK
       WHERE member.category EQ 0.17567323 TRANSACTION:
      ASSIGN member.memberName = 'New member name':U.
    END.
    ```

<a name="no--wait"></a><a name="3.3"></a>
  - [3.3](#no--wait) **No-wait**: When use NO-WAIT with NO-ERROR, always check whether record is LOCKED or not
    > Why? When you use NO-WAIT with NO-ERROR and record is locked, it also is not available. Check for AVAIABLE only will most likely cause undesirable outcome.

    ```openedge
    /* bad */
    FIND FIRST member EXCLUSIVE-LOCK
         WHERE member.id EQ 0.346544767 NO-WAIT NO-ERROR.
    IF NOT AVAILABLE member THEN
      CREATE member.
    /* good */
    FIND FIRST member EXCLUSIVE-LOCK
         WHERE member.id EQ 0.346544767 NO-WAIT NO-ERROR.
    IF LOCKED member THEN
      UNDO, THROW NEW Progress.Lang.AppError('Member record is current locked, please, try again later', 1000).
    ELSE IF NOT AVAILABLE member THEN
      CREATE member.
    ```

## Comments

## Performance

## Variables

<a name="record--locking"></a><a name="6.1"></a>
  - [6.1](#no--undo) **No-undo**: Always use NO-UNDO on all temp-tables and variables
    > Why? When you define variables, the AVM allocates what amounts to a record buffer for them, where each variable becomes a field in the buffer. There are in fact two such buffers, one for variables whose values can be undone when a transaction is rolled back and one for those that can't. There is extra overhead associated with keeping track of the before-image for each variable that can be undone, and this behavior is rarely needed.

    > Why not? If you need to be able to revert value of variable on UNDO

    ```openedge
    /* bad */
    DEFINE TEMP-TABLE ttMember
      FIELD member_name AS CHARACTER.
    DEFINE VARIABLE cMemberName AS CHARACTER.
    DEFINE PROPERTY MemberName AS CHARACTER
      GET.
      SET.

    /* good */
    DEFINE TEMP-TABLE ttMember NO-UNDO
      FIELD member_name AS CHARACTER.
    DEFINE VARIABLE cMemberName AS CHARACTER NO-UNDO.
    DEFINE PROPERTY MemberName AS CHARACTER NO-UNDO
      GET.
      SET.
    ```

## Naming Conventions
<a name="variable--case"></a><a name="7.1"></a>
  - [7.1](#variable-case) **Variable Case**: Use appropriate case when naming variable
  	
  	* When define variable use camelCase
  	```openedge
  	/* bad */
  	DEFINE VARIABLE Member_Name AS CHARACTER NO-UNDO.
  	DEFINE VARIABLE MEMBERNAME  AS CHARACTER NO-UNDO.
  	DEFINE VARIABLE membername  AS CHARACTER NO-UNDO.
  	DEFINE VARIABLE MeMbErNaMe  AS CHARACTER NO-UNDO.
  	/* good */
  	DEFINE VARIABLE cMemberName AS CHARACTER NO-UNDO.
	```
	
	* When define property use camelCase, unless you do it for GUI for .NET, then use PascalCase
	```openedge
	/* bad */
	DEFINE PROPERTY Member_Name AS CHARACTER NO-UNDO
	  GET.
	  SET.
	DEFINE PROPERTY MeMbErNaMe AS CHARACTER NO-UNDO
	  GET.
	  SET.
	
	/* good for GUI for .NET */
	DEFINE PROPERTY MemberName AS CHARACTER NO-UNDO
	  GET.
	  SET.

	/* good */
	DEFINE PROPERTY memberName AS CHARACTER NO-UNDO
	  GET.
	  SET.
	```

<a name="buffer--name"></a><a name="7.2"></a>
  - [7.2](#buffer--name) **Buffer Name**: When define buffer, use b/b# with original buffer name
	```openedge
	/* bad */
	DEFINE BUFFER MI1 FOR memberInfo.
	DEFINE BUFFER bu-memberInfo FOR memberInfo.
	DEFINE BUFFER memberInfoBuffer FOR memberInfo.
	DEFINE BUFFER bu-memberInfo-2 FOR memberInfo.
	
	/* good */
	DEFINE BUFFER bMemberInfo FOR memberInfo.
	DEFINE BUFFER b2MemberInfo FOR memberInfo.
	DEFINE BUFFER bProviderData FOR providerData.
	DEFINE BUFFER b_Field FOR _Field.
	DEFINE BUFFER bttMyTable FOR ttMyTable.
	```

<a name="variable--type"></a><a name="7.3"></a>
  - [7.3](#variable--type) **Variable Type**: Prefix variable name with it's type

    ```openedge
	 DEFINE VARIABLE cMemberName   AS CHARACTER    NO-UNDO.
	 DEFINE VARIABLE iMemberNumber AS INTEGER      NO-UNDO.
	 DEFINE VARIABLE dMemberId     AS DECIMAL      NO-UNDO.
	 DEFINE VARIABLE hMemberReq    AS HANDLE       NO-UNDO.
	 DEFINE VARIABLE dtStartDate   AS DATE         NO-UNDO.
	 DEFINE VARIABLE dtzEndDate    AS DATETIME-TZ  NO-UNDO.
	 DEFINE VARIABLE mMemberDoc    AS MEMPTR       NO-UNDO.
	 DEFINE VARIABLE rMemberKey    AS RAW          NO-UNDO.
	 DEFINE VARIABLE oMemberInfo   AS member.info  NO-UNDO.
	 DEFINE VARIABLE lcMemberNode  AS LONGCHAR     NO-UNDO.
	 ```
	 