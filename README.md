# OpenEdge Standards

## Table of Contents
1. [Objects](#objects)
1. [Error Handling](#error-handling)
1. [Data Access](#data-access)
1. [Comments](#comments)
1. [Performance](#performance)
1. [Variables](#variables)
1. [Naming Conventions](#naming-conventions)
1. [Dynamic Objects](#dynamic-objects)
1. [Other](#other)

## Objects

## Error Handling
<a name="no--error"></a><a name="2.1"></a>
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

<a name="no--error"></a><a name="2.2"></a>
  - [2.2](#routine-level) **BLOCK-LEVEL THROW** Always use BLOCK-LEVEL ON ERROR UNDO, THROW statement 
   > Why? It changes the default ON ERROR directive to UNDO, THROW for all blocks (from default UNDO, LEAVE or UNDO, RETRY)
   
   > Note: Use this parameter in legacy systems only. For new development use _-undothrow 2_ to set BLOCK-LEVEL ON ERROR UNDO, THROW everywhere 

   ````openedge
   /* bad (default ON ERROR directive used) */
   RUN internalProcedure.
   
   CATCH eExc AS Progress.Lang.AppError:
     /* this will never be executed */
   END.
   
   PROCEDURE internalProcedure:
     UNDO, THROW NEW Progress.Lang.AppError('Error String', 1000).
   END.
   ````

   ````openedge
   /* bad (routine-level doesn't cover FOR loops) */
   ROUTINE-LEVEL ON ERROR UNDO, THROW.
   RUN internalProcedure.
   
   CATCH eExc AS Progress.Lang.AppError:
     /* this will never be executed */
   END.
   
   PROCEDURE internalProcedure:
     FOR EACH memberRecord:
       UNDO, THROW NEW Progress.Lang.AppError('Error String', 1000).
     END.
   END.
   ````

<a name="no--error"></a><a name="2.2"></a>
  - [2.3](#catch-block) **CATCH/THROW** Use CATCH/THROW instead of classic error handling (NO-ERROR / ERROR-STATUS).
  > Why? One place to catch all errors. 
  
  > Why? No need to handle errors every time they occur. No need to return error as output parameter and then handle them on every call.

  ````openedge
  /* bad */
  RUN myCheck (OUTPUT cErrorMessage) NO-ERROR.
  IF ERROR-STATUS:ERROR THEN
    MESSAGE "Error: " + ERROR-STATUS:GET-MESSAGE(1).
  ELSE IF cErrorMessage GT '' THEN
    MESSAGE "Error: " + cErrorMessage.
  
  PROCEDURE myCheck:
    DEFINE OUTPUT PARAMETER opoStatusMessage AS CHARACTER NO-UNDO.
    
    IF NOT CAN-FIND (FIRST member) THEN
      DO:
        ASSIGN opoStatusMessage = 'Can not find member, try again'.
        RETURN.
      END.
  END.
 
  /* good (any application or system error will be catched by CATCH block) */
  RUN myCheck.
  
  CATCH eExc AS Progress.Lang.ProError:
      MESSAGE "Error: " + eExc:GetMessage(1).
  END.
  
  PROCEDURE myCheck:
    IF NOT CAN-FIND (FIRST member) THEN
      UNDO, THROW NEW Progress.Lang.AppError('Can not find member, try again', 1000).
    END.
  
  ````
   
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
  - [3.2](#exp--trans--scope) **Transaction Scope**: Always explicitly define the transaction scope

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
    > Why? When you use NO-WAIT with NO-ERROR and record is locked, it also is not available. Check for AVAILABLE only will most likely cause undesirable outcome.

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
<a name="use--for--first"></a><a name="9.2"></a>
  - [5.1](#use--for--first) **FOR FIRST/LAST**: Prefer to use FOR FIRST or FOR LAST instead of FIND FIRST/LAST
    > Why? FIND FIRST/LAST doesn't use multiple indexes (also there are issues with Oracle dataservers)
    
    > Why not? If you need to update record and want to check whether record is locked or not.

    ```openedge
    /* we have two single field indexes (benefitCodeIdx and memberIdIdx) */
    /* bad (only one index will be used) */
    FIND FIRST memberBenefit NO-LOCK
         WHERE memberBenefit.memberId    EQ 0.34521543
           AND memberBenefir.benefidCode EQ 'BLCA' NO-ERROR.
           
    /* good (both indexes will be used) */
    FOR FIRST memberBenefit NO-LOCK
        WHERE memberBenefit.memberId    EQ 0.34521543
          AND memberBenefir.benefidCode EQ 'BLCA':
    END.
    ```

<a name="define--buffer"></a><a name="9.2"></a>
  - [5.2](#define--buffer) **DEFINE BUFFER**: Define buffer for each DB buffer
    > Why? To avoid side-effects from buffer that may be used in internal procedures / functions
    > Why? To prevent locking issues which can happen when buffer is used in STATIC/SINGLETON methods or PERSISTENT procedures
    
    ```openedge
    /* bad (if this find was called from static/singleton class - record will stay locked) */
    METHOD PUBLIC CHARACTER getMember():
      FIND FIRST member EXSLUSIVE-LOCK.
      IF AVAILABLE member THEN
        DO:
          ASSIGN member.memberViewCount = member.memberViewCount - 1.
          RETURN member.id.
        END.
    END.
    
    /* good (if this find was called from static/singleton class - record will lock will be released) */
    METHOD PUBLIC CHARACTER getMember():
      DEFINE BUFFER bMember FOR member.
      FIND FIRST bMember EXSLUSIVE-LOCK.
      IF AVAILABLE bMember THEN
        DO:
          ASSIGN bMember.memberViewCount = bMember.memberViewCount - 1.
          RETURN bMember.id.
        END.
    END.
    
    ```



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
  	DEFINE VARIABLE membername  AS CHARACTER NO-UNDO.
  	DEFINE VARIABLE MeMbErNaMe  AS CHARACTER NO-UNDO.
  	/* good */
  	DEFINE VARIABLE cMemberName AS CHARACTER NO-UNDO.
	```

	* When define constants use UPPER_CASE
	
	```openedge
  	/* bad */ 
  	DEFINE PRIVATE PROPERTY lineseparator AS CHARACTER NO-UNDO INIT '|':U
  		GET.
  	DEFINE PRIVATE PROPERTY LineSeparator AS CHARACTER NO-UNDO INIT '|':U
  		GET.
  	DEFINE PRIVATE PROPERTY cLineSeparator AS CHARACTER NO-UNDO INIT '|':U
  		GET.
  	DEFINE PRIVATE PROPERTY LINESEPARATOR AS CHARACTER NO-UNDO INIT '|':U
  		GET.
  	/* good */
  	DEFINE PRIVATE PROPERTY LINE_SEPARATOR AS CHARACTER NO-UNDO INIT '|':U
  		GET.
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
  - [7.2](#buffer--name) **Buffer Name**: When define buffer, prefix with b
  
	```openedge
	/* bad */
	DEFINE BUFFER MI1 FOR memberInfo.
	DEFINE BUFFER bu-memberInfo FOR memberInfo.
	DEFINE BUFFER memberInfoBuffer FOR memberInfo.
	DEFINE BUFFER cMemberInfoBuffer FOR memberInfo.
	DEFINE BUFFER bu-memberInfo-2 FOR memberInfo.
	
	/* good */
	DEFINE BUFFER bMemberInfo FOR memberInfo.
	DEFINE BUFFER bMemberInfo2 FOR memberInfo.
	DEFINE BUFFER bProviderData FOR providerData.
	DEFINE BUFFER b_Field FOR _Field.
	DEFINE BUFFER bttMyTable FOR TEMP-TABLE ttMyTable.
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

<a name="variable--meaning"></a><a name="7.4"></a>
  - [7.4](#variable--meaning) **Meaningful Names**: Define variables with meaningful names (applicable to context), but avoid extra long names (use abbreviations when possible)
  	
  	```openedge
	/* bad */
	DEFINE VARIABLE cMI AS CHARACTER NO-UNDO.
	/* good */
	DEFINE VARIABLE cMemberInfo AS CHARACTER NO-UNDO.
	
	/* bad */
	DEFINE VARIABLE cNationalDrugCode AS CHARACTER NO-UNDO.
	/* good */
	DEFINE VARIABLE cNDC AS CHARACTER NO-UNDO.
	
	/* bad */
	DEFINE VARIABLE cNationalDrugCodeRequiredIdentification NO-UNDO.
	/* good */
	DEFINE VARIABLE cNDCReqId AS CHARACTER NO-UNDO.
	```

## Dynamic Objects
<a name="delete--objects"></a><a name="8.1"></a>
  - [8.1](#delete--objects) **Delete Dynamic Objects**: Always delete dynamic objects. Use FINNALY block to make sure object will be deleted.
    > Why? Progress garbage collector takes care of objects, but doesn't handle dynamic objects (BUFFERS, TABLES, QUERIES, PERSISTENT PROCEDURES and etc)
    
    ```openedge
    /* bad */
    PROCEDURE checkMember:
      DEFINE OUTPUT PARAMETER oplValidMember AS LOGICAL NO-UNDO.
      DEFINE VARIABLE hMemberBuffer AS HANDLE NO-UNDO.

      CREATE BUFFER hMemberBuffer FOR db + '.memberInfo'. 
      /* ... */
      ASSIGN oplValidMember = TRUE.
      RETURN.
    END.
    
    /* good */
    PROCEDURE checkMember:
      DEFINE OUTPUT PARAMETER oplValidMember AS LOGICAL NO-UNDO.
      DEFINE VARIABLE hMemberBuffer AS HANDLE NO-UNDO.

      CREATE BUFFER hMemberBuffer FOR db + '.memberInfo'.
      /* ... */
      ASSIGN oplValidMember = TRUE.
      RETURN.
      FINALLY:
        IF VALID-HANDLE(hMemberBuffer) THEN
          DELETE OBJECT hMemberBuffer.
      END.
    END.
    ```
# Dynamic Objects    
<a name="block--labels"></a><a name="9.1"></a>
  - [9.1](#block--labels) **Block Labels**: Always use block labels
    > Why? If you do not name a block, the AVM leaves the innermost iterating block that contains the LEAVE statement. The same is applicable to UNDO and NEXT statements. THis can cause unexpected behaviour

    ```openedge
    /* bad */
    DO TRANSACTION:
      FOR EACH memberBenefit:
        ...
        /* this will affect only current iteration */
        UNDO, LEAVE.
      END.
    END.
    
    /* good */
    UpdateMembersBlk:
    DO TRANSACTION:
      FOR EACH memberBenefit:
        ...
        /* this will undo the entire transaction and leave DO block */
        UNDO UpdateMembersBlk, LEAVE UpdateMembersBlk.
      END.
    END.
    
    ```
    
<a name="unnecessary-blocks"></a><a name="9.2"></a>
  - [9.2](#unnecessary-blocks) **Unnecessary Blocks**: Don't create unnecessary DO blocks

  ```openedge
  /* bad */
  IF NOT isValidMember(member.id) THEN
    DO:
      UNDO, THROW NEW Progress.Lang.AppError('Invalid Member', 1000).
    END.
  
  /* good */
  IF NOT isValidMember(member.id) THEN
    UNDO, THROW NEW Progress.Lang.AppError('Invalid Member', 1000).
  ```
  
<a name="comp--operators"></a><a name="9.3"></a>
  - [9.1](#comp--operators) **Comparison operators**: Use comparison operators: EQ(=), LT(<), LE(<=), GT(>), GE(>=), NE(<>) 
    > Why? It's easier to see/parse places whether we compare or assign values 

    ```openedge
    /* bad */
    IF memberDOB > 01/01/1980 THEN
    /* good */
    IF memberDOB GT 01/01/1980 THEN
    ```

<a name=""></a><a name="9.3"></a>
  - [9.1](#comp--operators) **Comparison operators**: Use comparison operators: EQ(=), LT(<), LE(<=), GT(>), GE(>=), NE(<>) 
    > Why? It's easier to see/parse places whether we compare or assign values 


  
