# MSSQL Examples
List of examples for SQL Server.<br/><br/>

## Try...Catch...Rollback
```
BEGIN TRY  
    BEGIN TRANSACTION;

    COMMIT TRANSACTION;  
END TRY  
BEGIN CATCH  
    SELECT
    ERROR_NUMBER() AS ErrorNumber  
        , ERROR_SEVERITY() AS ErrorSeverity  
        , ERROR_STATE() AS ErrorState  
        , ERROR_PROCEDURE() AS ErrorProcedure  
        , ERROR_LINE() AS ErrorLine  
        , ERROR_MESSAGE() AS ErrorMessage;  

    DECLARE @Message varchar(MAX) = ERROR_MESSAGE(),
				@Severity int = ERROR_SEVERITY(),
				@State smallint = ERROR_STATE()
 
	RAISERROR(@Message, @Severity, @State)
    ROLLBACK TRANSACTION;
END CATCH;
GO
```

## Delete all data and set the autonumber to 0 (When not available truncate)
```
DELETE FROM TABLENAME
DBCC CHECKIDENT ('DATABASENAME.dbo.TABLENAME',RESEED, 0)
```


## Running number
```
CREATE PROCEDURE [dbo].[usp_transfer_serial_range_generate]
(
	@TransferID INT
)
AS

BEGIN
SET NOCOUNT ON
BEGIN TRAN

-- Variables
DECLARE @TrnTransferItemValidationID AS INT
DECLARE @SerialFrom BIGINT, @SerialTo BIGINT
DECLARE @DateTimeNow DATETIME

-- Delete existing records
SELECT @TrnTransferItemValidationID = ID FROM trnTransferItemValidation WHERE TransferID = @TransferID AND IsActive = 1
DELETE FROM tmpTransferItemValidation WHERE TrnTransferItemValidationID IN (SELECT ID FROM trnTransferItemValidation WHERE TransferID = @TransferID)

-- Generate serial range
SELECT @SerialFrom = SerialFrom, @SerialTo = SerialTo FROM trnTransferItemValidation WHERE ID = @TrnTransferItemValidationID
SELECT @DateTimeNow = dbo.GetDateMY()

;WITH gen AS (
	SELECT @SerialFrom AS num
	UNION all
	SELECT num + 1 FROM gen WHERE num + 1 <= @SerialTo
)
INSERT INTO tmpTransferItemValidation(TrnTransferItemValidationID, ImeiSerial, IsValid)
SELECT	@TrnTransferItemValidationID, num, 0
FROM	gen
OPTION (maxrecursion 0)

COMMIT
END

GO
```
