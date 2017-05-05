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
1. [Code Styling](#code-styling)
1. [Other](#other)

## Objects

## Error Handling
<a name="no--error"></a><a name="2.1"></a>
  - [2.1](#no--error) **NO-ERROR**: Use NO-ERROR only when you expect an error to occur, and if you use it, handle error appropriately

    > Why? NO-ERROR suppresses errors, which can cause database inconsistency issues, memory leaks, infinite loops and more...

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

    /* good (ver 2) - classic error handling (split safe assignment from unsafe) */
    ASSIGN cMemberName   = 'ABC'.
    ASSIGN iMemberNumber = INTEGER("ABC") NO-ERROR.
    IF ERROR-STATUS:ERROR THEN
      DO:
        /* handle error here */
      END.
    ```

<a name="no--error"></a><a name="2.2"></a>
  - [2.2](#routine-level) **BLOCK-LEVEL THROW** Always use BLOCK-LEVEL ON ERROR UNDO, THROW statement

   > Why? It changes the default ON ERROR directive to UNDO, THROW for all blocks (from default UNDO, LEAVE or UNDO, RETRY)

   > Note: Use this parameter in legacy systems only. For new development use _-undothrow 2_ to set BLOCK-LEVEL ON ERROR UNDO, THROW everywhere

   > Note: Use in new modules or when you confident that the change on existing code is not going to break error handling

   ```openedge
   /* bad (default ON ERROR directive used) */
   RUN internalProcedure.
   
   CATCH eExc AS Progress.Lang.AppError:
     /* this will never be executed */
   END.
   
   PROCEDURE internalProcedure:
     UNDO, THROW NEW Progress.Lang.AppError('Error String', 1000).
   END.
   ```

   ```openedge
   /* bad (routine-level doesn't cover FOR loops) */
   ROUTINE-LEVEL ON ERROR UNDO, THROW.
   RUN internalProcedure.
   
   CATCH eExc AS Progress.Lang.AppError:
     /* this will never be executed */
   END.
   
   PROCEDURE internalProcedure:
     FOR EACH bMemberRecord NO-LOCK:
         IF bMemberRecord.memberDOB LT 01/01/1910 THEN
             UNDO, THROW NEW Progress.Lang.AppError('Found member with invalid DOB', 1000).
     END.
   END.
   ```

<a name="no--error"></a><a name="2.3"></a>
  - [2.3](#catch-block) **CATCH/THROW** Use CATCH/THROW instead of classic error handling (NO-ERROR / ERROR-STATUS).

  > Why? One place to catch all errors.
  
  > Why? No need to handle errors every time they occur. No need to return error as output parameter and then handle them on every call.

  ```openedge
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
 
  /* good (any application or system error will be caught by CATCH block) */
  RUN myCheck.
  
  CATCH eExc AS Progress.Lang.ProError:
      MESSAGE "Error: " + eExc:GetMessage(1).
  END.
  
  PROCEDURE myCheck:
    IF NOT CAN-FIND (FIRST member) THEN
      UNDO, THROW NEW Progress.Lang.AppError('Can not find member, try again', 1000).
    END.
  
  ```

<a name="catch--many"></a><a name="2.4"></a>
  - [2.4](#catch-many) **CATCH MANY** Use multiple catch blocks if you need to handle errors differently based on error type (only if you need to handle errors differently)

  ```openedge
  /* bad (one catch - multiple error types) */
  ASSIGN iMemberId = INTEGER(ipcParsedMemberId).
  FIND FIRST member NO-LOCK
       WHERE memberId = iMemberId NO-ERROR.
  IF NOT AVAILABLE member THEN
    UNDO, THROW NEW Progress.Lang.AppError('Invalid Member Id', 1000).
  
  CATCH eExc AS Progress.Lang.Error:
    IF TYPE-OF(eExc, Progress.Lang.AppError) THEN
      RETURN 'Invalid Application Error: ' + eExc:GetMessage(1).
    ELSE
      RETURN 'Invalid System Error: ' + eExc:GetMessage(1).
  END.
    
  /* good */
  ASSIGN iMemberId = INTEGER(ipcParsedMemberId).
  FIND FIRST member NO-LOCK
       WHERE memberId = iMemberId NO-ERROR.
  IF NOT AVAILABLE member THEN
    UNDO, THROW NEW Progress.Lang.AppError('Invalid Member Id', 1000).
  
  CATCH eExcApp AS Progress.Lang.AppError:
    RETURN 'Invalid Application Error: ' + eExcApp:GetMessage(1).
  END.
  
  CATCH eExcSys AS Progress.Lang.Error:
    RETURN 'Invalid System Error: ' + eExcSys:GetMessage(1).
  END.
  ```

  ```openedge
  /* bad (multiple catch blocks - the same error handling) */
  FIND FIRST member NO-LOCK
       WHERE memberId = 123 NO-ERROR.
  IF NOT AVAILABLE member THEN
    UNDO, THROW NEW Progress.Lang.AppError('Invalid Member Id', 1000).
    
  CATCH eExcApp AS Progress.Lang.AppError:
    RETURN 'Error: ' + eExcApp:GetMessage(1).
  END CATCH.
    
  CATCH eExcSys AS Progress.Lang.Error:
    RETURN 'Error: ' + eExcSys:GetMessage(1).
  END CATCH.
    
  /* good */
  FIND FIRST member NO-LOCK
       WHERE memberId = 123 NO-ERROR.
  IF NOT AVAILABLE member THEN
    UNDO, THROW NEW Progress.Lang.AppError('Invalid Member Id', 1000).
    
  CATCH eExc AS Progress.Lang.Error:
    RETURN 'Error: ' + eExc:GetMessage(1).
  END CATCH.
  ```

<a name="no--error"></a><a name="2.5"></a>
  - [2.5](#rethrow) **RE-THROW** Re-throw errors manually only if you need to do extra processing (like logging, or converting general error type to more specific) before error is thrown to upper level
  
    > Why? Every procedure/class is supposed to change the default ON ERROR directive, forcing AVM to throw errors to upper level automatically

    ```openedge
    /* bad */
    ASSIGN iMemberId = INTEGER('ABC_123').
        
    CATCH eExc AS Progress.Lang.ProError:
      UNDO, THROW eExc.
    END CATCH.
        
    /* good (convert General error into ParseError) */
    ASSIGN iMemberId = INTEGER('ABC_123').
        
    CATCH eExc AS Progress.Lang.ProError:
      UNDO, THROW NEW Mhp.Errors.ParseError(eExc:GetMessage(1), eExc:GetNumber(1)).
    END CATCH.
        
    /* good (write log message and throw error - use only at top most level) */
    ASSIGN iMemberId = INTEGER('ABC_123').
        
    CATCH eExc AS Progress.Lang.ProError:
      logger:error(eExc).
      UNDO, THROW NEW Mhp.Errors.ParseError(eExc:GetMessage(1), eExc:GetNumber(1)).
    END CATCH.
    ```

## Data Access

<a name="record--locking"></a><a name="3.1"></a>
  - [3.1](#record--locking) **Never use SHARE-LOCK**: Always use either NO-LOCK or EXCLUSIVE-LOCK

    > Why? If you don't specify locking mechanism, SHARE-LOCK is used. NO-LOCK has better performance over SHARE-LOCK. Other users can't obtain EXCLUSIVE-LOCK on record that is SHARE locked

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
  - [3.2](#exp--trans--scope) **Transaction Scope**: Always explicitly define the transaction scope and strong scope applicable buffer

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
    DO FOR provider TRANSACTION:
      FIND FIRST provider EXCLUSIVE-LOCK
           WHERE provider.id EQ 0.657532547 NO-ERROR.
      IF AVAILABLE provider THEN
        ASSIGN provider.name = 'New Provider':U.
    END.
    /* ... some code... */
    FOR EACH bMember NO-LOCK
       WHERE bMember.category EQ 0.17567323 TRANSACTION:
      FIND FIRST bMember2 EXCLUSIVE-LOCK
           WHERE ROWID(bMember2) EQ ROWID(bMember).
      ASSIGN bMember2.memberName = 'New member name':U.
    END.
    ```

<a name="no--wait"></a><a name="3.3"></a>
  - [3.3](#no--wait) **No-wait**: When use NO-WAIT with NO-ERROR, always check whether record is LOCKED or not

    > Why? When you use NO-WAIT with NO-ERROR and record is locked, it also is not available. Checking only for AVAILABLE, will most likely cause undesirable outcome.

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
      UNDO, THROW NEW Progress.Lang.AppError('Member record is currently locked, please, try again later', 1000).
    ELSE IF NOT AVAILABLE member THEN
      CREATE member.
    ```

<a name="no--recid"></a><a name="3.4"></a>
  - [3.4](#no--recid) **No RECID**: Don't use RECID, use ROWID. Don't store ROWID or RECID in database, use surrogate keys

    > Why ROWID? RECID supported only for backward compatibility
    
    > Why don't store? ROWID and RECID change after dump and load process. The same ROWID/RECID can be found in the same table (when multi-tenancy or data-partitioning is used)

    ```openedge
    /* good */
    FIND FIRST member NO-LOCK
         WHERE ROWID(member) EQ rMemberRowId NO-ERROR.
    ```

<a name="use--canfind"></a><a name="3.5"></a>
  - [3.5](#use--canfind) **CAN-FIND**: Use CAN-FIND instead of FIND FIRST/LAST or FOR FIRST/LAST if all you need is only to check that record exists
    
    > Why not? If multiple indexes needed to effectively find a record (use FOR FIRST/LAST in this case).
    
    ```openedge
    /* bad */
    FIND FIRST member NO-LOCK
         WHERE ROWID(member) EQ rMemberRowId NO-ERROR.
    IF AVAIABLE member THEN
        RETURN TRUE.
    ELSE
        RETURN FALSE.
        
    /* good */
    RETURN CAN-FIND (FIRST member NO-LOCK
                     WHERE ROWID(member) EQ rMemberRowId).
    ```

<a name="use--table--name"></a><a name="3.6"></a>
  - [3.6](#use--table--name) **TABLE-NAME**: Always qualify table name for field name
    
    ```openedge
    /* bad */
    FIND FIRST member NO-LOCK NO-ERROR.
    IF AVAIABLE member THEN
        RETURN memberId.
        
    /* good */
    FIND FIRST member NO-LOCK NO-ERROR.
    IF AVAIABLE member THEN
        RETURN member.memberId.
    ```

<a name="use--index"></a><a name="3.7"></a>
  - [3.7](#use--index) **USE-INDEX**: Avoid using USE-INDEX statement. Use TABLE-SCAN if you need to read entire table.

    >Why? AVM automatically selects the most appropriate index
    
    >Why not? USE-INDEX can be used to force display order (applicable to temp-tables)

    >Why not? In case you need to process records in batches and index selection can't be consistently enforced by WHERE clause (REPOSITION TO ROW-ID -> NEXT/PREV)

    ```openedge
    /* bad */
    FOR EACH member NO-LOCK
       WHERE member.DOB > 01/01/1982 USE-INDEX memberSSNIdx:
    END.
        
    /* good (let AVM to choose index automatically). Use temp-table with appropriate index, if you need to have different order in UI */
    FOR EACH member NO-LOCK
       WHERE member.DOB > 01/01/1982:
    END.
        
    /* good (if the whole table scan is needed) */
    FOR EACH member NO-LOCK TABLE-SCAN:
    END.
        
    /* good (AVM will use index, if available, to prepare sorted result) */
    /* IMPORTANT: Be careful when combining WHERE and BY statement (use XREF to see used index) */
    FOR EACH member NO-LOCK BY member.firstName:
    END.
    ```

## Comments
<a name="comm-header"></a><a name="4.1"></a>
  - [4.1](#comm-header) **Header comments**: Every class or external procedure has to have header aligned to ABLDocs format

    > Note: Put note in header if procedure/class is used by external system/service (exposed as REST service)

    ```openedge
    /*------------------------------------------------------------------------
        File        : getMinUserInfo.p
        Purpose     : Use to find and return user information
        Syntax      : RUN getMinUserInfo.p (userId, OUTPUT dsUserInfo).
        Description : This procedure finds user information and returns it in ProDataSet
        Author(s)   : Aliaksandr Tarasevich
        Created     : <pre>Tue Nov 19 21:16:10 CET</pre>
        Notes       : Use to only retrieve minimal user information, otherwise, use getAllUserInfo.p
                      Used by external systems (REST/SOAP)
      ----------------------------------------------------------------------*/
    ```
<a name="comm--proc--func"></a><a name="4.2"></a>
  - [4.2](#comm--proc--func) **Internal comments**: Use comments in internal procedures, functions, methods aligned to ABLDocs format

    ```openedge
    /*------------------------------------------------------------------------------
     Purpose: Find a member and return TRUE if this member is active
     Notes: Doesn't take temporary member status into account
     @param ipdMemberId - member Id
     @return Is member active?
    ------------------------------------------------------------------------------*/
    METHOD PUBLIC LOGICAL isMemberActive (INPUT ipdMemberId AS DECIMAL):
    END METHOD.
        
    PROCEDURE isMemberActive:
      /*------------------------------------------------------------------------------
       Purpose: Find a member and return TRUE if this member is active
       Notes: Doesn't take temporary member status into account
       @param ipdMemberId - member Id
       @return Is member active?
      ------------------------------------------------------------------------------*/
    
      DEFINE INPUT  PARAMETER ipdMemberId AS DECIMAL NO-UNDO.
      DEFINE OUTPUT PARAMETER oplIsActive AS LOGICAL NO-UNDO.
    END PROCEDURE.
        
    FUNCTION isMemberActive RETURNS LOGICAL (INPUT ipdMemberId AS DECIMAL):
      /*------------------------------------------------------------------------------
       Purpose: Find a member and return TRUE if this member is active
       Notes: Doesn't take temporary member status into account
       @param ipdMemberId - member Id
       @return Is member active?
      ------------------------------------------------------------------------------*/
    END FUNCTION.
    ```

<a name="class--props"></a><a name="4.3"></a>
  - [4.3](#class--props) **Class Properties**: Use comments for properties aligned to ABLDocs format

    ```openedge
    /*
       Indicates whether document was previously loaded or not
     */
    DEFINE PUBLIC PROPERTY docLoaded AS LOGICAL NO-UNDO
      GET.
      PROTECTED SET.
    ```

<a name="no--commented--code"></a><a name="4.4"></a>
  - [4.4](#no--commented--code) **No commented code**: Don't leave unused code commented - delete.

    > Why? It makes code less readable and causes unused code to be picked up when developer tries to search code snip in codebase

    > Why? Developers must use version control system to keep track of changes

    > Note: If you commented out procedure/function/method calls, fine whether commented procedure/function/method is used in other places, if not - delete it

    ```openedge
    /* bad */

    /* removed as part of fix for PL-43516674 */
    /* IF member.memberId EQ dInvMemberId THEN
         DO:
           ...
         END. */

    /* temporary remove as part of N project */
    /* RUN makeMemberAsInvalid. */
    ```


## Performance
<a name="use--for--first"></a><a name="5.1"></a>
  - [5.1](#use--for--first) **FOR FIRST/LAST**: Prefer to use FOR FIRST or FOR LAST instead of FIND FIRST/LAST

    > Why? FIND FIRST/LAST doesn't use multiple indexes (also there are issues with Oracle dataservers)
    
    > Why not? Use if you need to update record and want to check whether record is locked or not.

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

<a name="define--buffer"></a><a name="5.2"></a>
  - [5.2](#define--buffer) **DEFINE BUFFER**: Define buffer for each DB buffer

    > Why? To avoid side-effects from buffer that may be used in internal procedures / functions

    > Why? To prevent locking issues which can happen when buffer is used in STATIC/SINGLETON methods or PERSISTENT procedures

    > Note: Define buffer as close as possible to a place it's going to be use (internal procedure or method). Try to avoid using globally defined buffers

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

<a name="by--reference"></a><a name="5.3"></a>
  - [5.3](#by--reference) **BY-REFERENCE**: Always use BY-REFERENCE or BIND when passing temp-table or dataset to procedures/methods

    > Why? By default AVM clones temp-table / dataset when it's passed as parameter (BY-VALUE)

    > Why not? If you want to merge result manually or target procedure changes cursor positions in temp-table

    ```openedge

    /* bad */
    RUN getMemberInfo.p ( OUTPUT DATASET dsMemberInfo ).

    /* good */
    RUN getMemberInfo.p ( OUTPUT DATASET dsMemberInfo BY-REFERENCE ).
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
    DEFINE INPUT PARAMETER ipcMemberName AS CHARACTER.
    DEFINE PROPERTY MemberName AS CHARACTER
      GET.
      SET.

    /* good */
    DEFINE TEMP-TABLE ttMember NO-UNDO
      FIELD member_name AS CHARACTER.
    DEFINE VARIABLE cMemberName AS CHARACTER NO-UNDO.
    DEFINE INPUT PARAMETER ipcMemberName AS CHARACTER NO-UNDO.
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
	 DEFINE VARIABLE tEndDate      AS DATETIME     NO-UNDO.
	 DEFINE VARIABLE tzEndDate     AS DATETIME-TZ  NO-UNDO.
	 DEFINE VARIABLE mMemberDoc    AS MEMPTR       NO-UNDO.
	 DEFINE VARIABLE rMemberKey    AS RAW          NO-UNDO.
	 DEFINE VARIABLE oMemberInfo   AS member.info  NO-UNDO.
	 DEFINE VARIABLE lcMemberNode  AS LONGCHAR     NO-UNDO.
	 ```

<a name="input--prefix"></a><a name="7.4"></a>
  - [7.4](#input--prefix) **Prefix input/output variable**: Prefix parameter variable with input/output type (ip - INPUT, op - OUTPUT, oip - INPUT-OUTPUT)

    ```openedge
	 DEFINE INPUT  PARAMETER ipcMemberName      AS CHARACTER    NO-UNDO.
	 DEFINE OUTPUT PARAMETER opiMemberNumber    AS INTEGER      NO-UNDO.
	 DEFINE INPUT-OUTPUT PARAMETER oipdMemberId AS DECIMAL      NO-UNDO.

	 METHOD PUBLIC LOGICAL checkMemberReq (INPUT         iphMemberReq  AS HANDLE).
	 METHOD PUBLIC LOGICAL checkMemberReq (OUTPUT        opdtStartDate AS DATE).
	 METHOD PUBLIC LOGICAL checkMemberReq (INPUT-OUTPUT  ioptEndDate   AS DATETIME).
	 ```

<a name="input--prefix"></a><a name="7.5"></a>
  - [7.5](#input--prefix) **Prefix temp-table/prodataset**: Put prefix on temp-tables (tt, bi) and datasets (ds)

    ```openedge
    /* bad */
    DEFINE TEMP-TABLE tbMember...
    DEFINE TEMP-TABLE ttblMember...
    DEFINE TEMP-TABLE ttMember NO-UNDO BEFORE-TABLE bttMember...
    DEFINE DATASET pdsMember...
    DEFINE DATASET pMember...

    /* good */
    DEFINE TEMP-TABLE ttMember NO-UNDO BEFORE-TABLE bittMember...
    DEFINE DATASET dsMember...
    ```

<a name="variable--meaning"></a><a name="7.6"></a>
  - [7.6](#variable--meaning) **Meaningful Names**: Define variables with meaningful names (applicable to context), but avoid extra long names (use abbreviations when possible)
  	
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
  - [8.1](#delete--objects) **Delete Dynamic Objects**: Always delete dynamic objects. Use FINALLY block to make sure object will be deleted.

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

<a name="mem--pointer"></a><a name="8.2"></a>
  - [8.2](#mem--pointer) **MEMPTR**: Always deallocate memory allocated for MEMPTR.

    > Why? Progress garbage collector doesn't take care of memory pointers.

    > Why not? If you need to pass memory pointer to caller procedure/method as output parameter. Then make sure you clean up there.

    ```openedge
    /* bad (cause memory leak) */
    PROCEDURE parseFile:
      DEFINE VARIABLE mBlob AS MEMPTR NO-UNDO.
      ...
      COPY-LOB FILE 'path-to-file' TO mBlob.
      ...
    END.

    /* good */
    PROCEDURE parseFile:
      DEFINE VARIABLE mBlob AS MEMPTR NO-UNDO.
      ...
      COPY-LOB FILE 'path-to-file' TO mBlob.
      ...
      FINALLY:
        ASSIGN SET-SIZE(mBlob) = 0.
      END.
    END.

    /* good */
    RUN parseFile(OUTPUT mLoadedFile).

    FINALLY:
      ASSIGN SET-SIZE(mLoadedFile) = 0.
    END.

    PROCEDURE loadFile:
      DEFINE OUTPUT PARAMETER opmBlob AS MEMPTR NO-UNDO.
      ...
      COPY-LOB FILE 'path-to-file' TO opmBlob.
      ...
    END.

    ```

# Code Styling
<a name="unnecessary-blocks"></a><a name="9.1"></a>
  - [9.1](#unnecessary-blocks) **Unnecessary Blocks**: Don't create unnecessary DO blocks

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

<a name="comp--operators"></a><a name="9.2"></a>
  - [9.2](#comp--operators) **Comparison operators**: Use comparison operators: EQ(=), LT(<), LE(<=), GT(>), GE(>=), NE(<>) 

    > Why? It's easier to see/parse places where we compare or assign values

    ```openedge
    /* bad */
    IF memberDOB > 01/01/1980 THEN
        
    /* good */
    IF memberDOB GT 01/01/1980 THEN
    ```

<a name="same--line-dot"></a><a name="9.3"></a>
  - [9.3](#same--line-dot) **Dot/Colon Same Line** Put dot and colon on the same line:

    ```openedge
    /* bad */
    IF memberDOB > 01/01/1980 THEN
      RETURN memberDOB
      .
      
    /* bad */
    IF memberDOB > 01/01/1980 THEN
      RETURN memberDOB
    .

    /* bad */
    FOR EACH memberInfo NO-LOCK
      :

    /* bad */
    FOR EACH memberInfo NO-LOCK
       WHERE memberInfo.memberId EQ 0.54764767
       :

    /* good */
    IF memberDOB > 01/01/1980 THEN
      RETURN memberDOB.

    /* good */
    FOR EACH memberInfo NO-LOCK:

    /* good */
    FOR EACH memberInfo NO-LOCK
       WHERE memberInfo.memberId EQ 0.54764767:
    ```

<a name="blk--indentation"></a><a name="9.4"></a>
  - [9.4](#blk--indentation) **Block Indentation**: Use correct block indentation: DO statements on next line with 2 characters, otherwise 4 characters. __Make sure you configured Tab policy in Eclipse to use Spaces only (4 spaces per tab)__

    ```openedge
    /* bad */
    IF memberDOB > 01/01/1980 THEN ASSIGN RETURN memberName.
        
    /* bad */
    IF memberDOB > 01/01/1980 THEN DO:
      RETURN memberName.
    END.
        
    /* bad */
    IF memberDOB > 01/01/1980 THEN DO:
    IF memberCode = 'OBC' THEN DO:
    END.
    END.
        
    /* bad */
    IF memberDOB > 01/01/1980 THEN
      RETURN memberName.
      ELSE 
        RETURN memberCode.
    
            
    /* good (new line + tab) */
    IF memberDOB GT 01/01/1980 THEN
        RETURN memberName.
        
    /* good (new line + do (2 chars) + new line + tab) */
    IF memberDOB GT 01/01/1980 THEN
      DO:
        ...
        RETURN memberName.
      END.
      
    ```

<a name="where--position"></a><a name="9.5"></a>
  - [9.5](#where--position) **WHERE NEW LINE**: Always put WHERE and AND/OR on next line.

    ```openedge
    /* bad */
    FOR EACH memberInfo FIELDS(birthDate gender) NO-LOCK WHERE memberInfo.birthDate GT 01/01/1920 AND memberInfo.gender EQ 'M':
    
    /* good */
    FOR EACH memberInfo FIELDS(birthDate gender) NO-LOCK
       WHERE memberInfo.birthDate GT 01/01/1920
         AND memberInfo.gender    EQ 'M'
    ```

<a name="method--params"></a><a name="9.6"></a>
  - [9.6](#method--params) **Parameters**: Put first method/function/procedure parameter on the same line. If method has more than 3 parameters, put every parameter on new line (aligned to first parameter)

    ```openedge
    /* bad */
    RUN loadFile (
        cFileName,
        cFileType, OUTPUT mFileBody).

    /* bad */
    RUN loadFile (
        cFileName, cFileType, OUTPUT mFileBody
        ).

    /* bad */
    ASSIGN mFileBody =
        loadFile (cFileName, cFileType, OUTPUT mFileBody).

    /* good */
    RUN loadFile (cFileName, cFileType, OUTPUT mFileBody).

    /* good */
    RUN loadFile (INPUT cFileName,
                  INPUT cFileType,
                  INPUT cLoadCodePage,
                  OUTPUT mFileBody).

    /* good */
    ASSIGN mFileBody = loadFile (cFileName, cFileType).

    ```

<a name="if--parens"></a><a name="9.7"></a>
  - [9.7](#if--parens) **If Parentheses**: Always use parentheses when have AND and OR conditions or use IF in ASSIGN statement

    > Why? Even though precedence order is known, some people forget it or it gets mixed.

    ```openedge

    /* good */
    IF (isValidMember OR showAll) AND (memberDOB < 01/01/1982 OR memberStatus = 'A') THEN
      ...

    /* bad (cause unexpected behaviour - last name will be only used if member doesn't have first name) */
    ASSIGN cMemberFullName = IF cMemberFirstName GT '' THEN cMemberFirstName ELSE '' + ' ' + cMemberLastName.

    /* good */
    ASSIGN cMemberFullName = (IF cMemberFirstName GT '' THEN cMemberFirstName ELSE '') + ' ' + cMemberLastName.
    ```

<a name="single-quotes"></a><a name="9.8"></a>
  - [9.8](#single-quotes) **Single Quotes**: Use single quotation marks when working with string constants

    ```openedge
    /* bad */
    ASSIGN cMemberInfo = "Some Info".
        
    /* good */
    ASSIGN cMemberInfo = 'Some Info'.
    ```

<a name="end--with-type"></a><a name="9.9"></a>
  - [9.9](#end--with-type) **End with type**: Always specify what is end used for (PROCEDURE, CONSTRUCTOR, DESTRUCTOR, METHOD or FUNCTION)

    ```openedge
    /* bad */
    PROCEDURE checkMember:
    END.

    FUNCTION checkMember RETURNS LOGICAL():
    END.

    CONSTRUCTOR Member:
    END.

    DESTRUCTOR Member:
    END.

    METHOD PUBLIC LOGICAL checkMember():
    END.

    /* good */
    PROCEDURE checkMember:
    END PROCEDURE.

    FUNCTION checkMember RETURNS LOGICAL():
    END FUNCTION.

    CONSTRUCTOR Member:
    END CONSTRUCTOR.

    DESTRUCTOR Member:
    END DESTRUCTOR.

    METHOD PUBLIC LOGICAL checkMember():
    END METHOD.
    ```

<a name="methods--out--return"></a><a name="9.10"></a>
  - [9.10](#methods--out--return) **Consistent method/function return**: Either return value or use output parameters (don't mix) when working with methods / functions

    ```openedge
    /* bad */
    ASSIGN lValidMember = oMemberInfo:getMemberInfo(iMemberId, OUTPUT cMemberName, OUTPUT dMemberDOB).

    /* good */
    oMemberInfo:getMemberInfo(iMemberId, OUTPUT lValidMember, OUTPUT cMemberName, OUTPUT dMemberDOB).

    /* good (split into couple calls) */
    IF oMemberInfo:isValidMember(iMemberId) THEN
      oMemberInfo:getMemberInfo(iMemberId, OUTPUT cMemberName, OUTPUT dMemberDOB).

    ```

# Other
<a name="block--labels"></a><a name="10.1"></a>
  - [10.1](#block--labels) **Block Labels**: Always use block labels

    > Why? If you do not name a block, the AVM leaves the innermost iterating block that contains the LEAVE statement. The same is applicable to UNDO and NEXT statements. THis can cause unexpected behavior

    ```openedge
    /* bad */
    DO TRANSACTION:
      FOR EACH memberBenefit:
        ...
        /* this will affect current iteration only */
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
    
<a name="assign--statement"></a><a name="10.2"></a>
  - [10.2](#assign--statement) **Assign Statement**: Always use ASSIGN statement (even on single assignments)

    > Why? This method allows you to change several values with minimum I/O processing. Otherwise, the AVM re-indexes records at the end of each statement that changes the value of an index component.
    
    ```openedge
    ASSIGN member.dob  = 01/01/1980
           member.name = 'John'
           member.ssn  = '000-00-0000' WHEN lKeepSSN
           member.mid  = IF NOT lKeepSSN THEN '111' ELSE '000-00-0000'.
    ```  
<a name="use--substitute"></a><a name="10.3"></a>
  - [10.3](#use--substitute) **Use Substitute**: Use SUBSTITUTE to concatenate multiple values

    > Why? If you try to concatenate values and one of the values is ? the entire result becomes ?, which is in most cases an undesirable result.
    
    ```openedge
    ASSIGN cMemberName = 'John'
           dMemberDOB  = 01/01/1980
           cMemberAge  = ?.
    /* bad - show ? */
    MESSAGE cMemberName + STRING(dMemberDOB) + cMemberAge VIEW-AS ALERT-BOX.
    
    /* good - show 'John 01/01/1980 ?' */
    MESSAGE SUBSTITUTE('&1 &2 &3', cMemberName, dMemberDOB, cMemberAge).
    ```  