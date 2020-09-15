# T-SQL Examples
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


## Sample Stored Procedure
```
CREATE PROCEDURE [dbo].[usp_transfer_serial_range_confirm]
(
	@TransferID INT
)
AS

-- Variables
DECLARE @TrnTransferItemValidationID INT
DECLARE @StatusTransferID INT, @StatusInStockID INT, @StatusApproveID INT
DECLARE @BizUnitID INT, @WHCenterID INT, @AccessID INT
DECLARE @DateTimeNow DATETIME
DECLARE @IsAllValid BIT
DECLARE @ErrorMessage NVARCHAR(255) 

DECLARE @StockBalSummary TABLE
(
	ModelID INT,
	ColourID INT,
	TypeID INT,
	CenterID INT,
	InStock_StkBalID INT,
	Transfer_StkBalID INT,
	Qty INT
)

DECLARE @WHStockBalSummary TABLE
(
	ModelID INT,
	ColourID INT,
	TypeID INT,
	InStock_StkBalID INT,
    Transfer_StkBalID INT,
	Qty INT
)

-- EXEC SP
EXEC usp_transfer_serial_range_generate @TransferID
EXEC usp_transfer_serial_range_validate @TransferID, @ErrorMessage OUTPUT

-- Check error
IF @ErrorMessage IS NOT NULL
BEGIN
	--RAISERROR (15600,-1,-1, @ErrorMessage);  
	PRINT('++++++++++++');
	PRINT(@ErrorMessage);
	PRINT('++++++++++++');
	RETURN;
END


BEGIN
SET NOCOUNT ON
BEGIN TRAN


-- Variable assignment
SELECT @TrnTransferItemValidationID = ID FROM trnTransferItemValidation WHERE TransferID = @TransferID AND IsActive = 1
SELECT @DateTimeNow = dbo.GetDateMY()
SELECT @StatusInStockID = ID FROM refStatus WHERE Active = 1 AND Code = 'I'
SELECT @StatusApproveID = ID FROM refStatus WHERE Active = 1 AND Code = 'AP'
SELECT @StatusTransferID = ID FROM refStatus WHERE Active = 1 AND Code = 'T'

SELECT  @BizUnitID = BizUnitID, @AccessID = AccessID
FROM	trnTransfer
WHERE	ID = @TransferID

SELECT  @WHCenterID = ISNULL(ParentID, 0)
FROM    refCenter 
WHERE   ID = (SELECT FromCenterID FROM trnTransfer WHERE ID = @TransferID)  

INSERT INTO @StockBalSummary
		    (ModelID, ColourID, TypeID, CenterID, InStock_StkBalID, Qty)
SELECT  	c.ModelID, c.ColourID, c.TypeID, c.CenterID, c.StkBalID, COUNT(1)
FROM    	trnTransferItemValidation a
		    JOIN tmpTransferItemValidation b ON b.TrnTransferItemValidationID = a.ID
		    JOIN tblItem c ON c.ID = b.ItemID
WHERE   	a.ID = @TrnTransferItemValidationID
GROUP BY    c.ModelID, c.ColourID, c.TypeID, c.CenterID, c.StkBalID

INSERT INTO @WHStockBalSummary
		    (ModelID, ColourID, TypeID, Qty)
SELECT      ModelID, ColourID, TypeID, SUM(Qty)
FROM	    @StockBalSummary
GROUP BY    ModelID, ColourID, TypeID

-- Create Stock Bal if not found 
INSERT INTO tblStkBalance
		    (BizUnitID, CenterID, ModelID, ColourID, TypeID, StatusID, Qty, ReserveQty, CreateDT, AccessID)
select	    @BizUnitID, a.CenterID, a.ModelID, a.ColourID, a.TypeID, @StatusTransferID, 0, 0, @DateTimeNow, @AccessID
from	    @StockBalSummary a
LEFT JOIN   tblStkBalance b ON
			b.ModelID = a.ModelID
			AND b.ColourID = a.ColourID
			AND b.TypeID = a.TypeID				
			AND b.CenterID = a.CenterID
			AND b.BizUnitID = @BizUnitID
			AND b.StatusID = @StatusTransferID
WHERE		b.ID IS NULL

-- Create WH Stock Bal if not found 
IF (@WHCenterID > 0)
BEGIN
INSERT INTO tblStkBalance
		    (BizUnitID, CenterID, ModelID, ColourID, TypeID, StatusID, Qty, ReserveQty, CreateDT, AccessID)
SELECT	    @BizUnitID, @WHCenterID, a.ModelID, a.ColourID, a.TypeID, @StatusTransferID, 0, 0, @DateTimeNow, @AccessID
FROM	    @WHStockBalSummary a
LEFT JOIN   tblStkBalance b ON
			b.ModelID = a.ModelID
			AND b.ColourID = a.ColourID
			AND b.TypeID = a.TypeID				
			AND b.CenterID = @WHCenterID
			AND b.BizUnitID = @BizUnitID
			AND b.StatusID = @StatusTransferID
WHERE	b.ID IS NULL
END

-- Map Transfer StkBalID
update	a
SET		a.Transfer_StkBalID = b.ID
FROM	@StockBalSummary a
		JOIN tblStkBalance b ON
			b.ModelID = a.ModelID
			AND b.ColourID = a.ColourID
			AND b.TypeID = a.TypeID				
			AND b.CenterID = a.CenterID
			AND b.BizUnitID = @BizUnitID
			AND b.StatusID = @StatusTransferID

-- Map WH StkBalID
IF (@WHCenterID > 0)
BEGIN
UPDATE	a
SET		a.InStock_StkBalID = b.ID,
		a.Transfer_StkBalID = c.ID
FROM	@WHStockBalSummary a
		JOIN tblStkBalance b ON
			b.ModelID = a.ModelID
			AND b.ColourID = a.ColourID
			AND b.TypeID = a.TypeID				
			AND b.CenterID = @WHCenterID
			AND b.BizUnitID = @BizUnitID
			AND b.StatusID = @StatusInStockID
		JOIN tblStkBalance c ON
			c.ModelID = a.ModelID
			AND c.ColourID = a.ColourID
			AND c.TypeID = a.TypeID				
			AND c.CenterID = @WHCenterID
			AND c.BizUnitID = @BizUnitID
			AND c.StatusID = @StatusTransferID
END

-- Update tblItem
UPDATE	a
SET		a.StkBalID = c.Transfer_StkBalID
		, StatusID = @StatusTransferID
		, TransferID = @TransferID
		, LastUpdateDT = @DateTimeNow
		, AccessID = @AccessID
FROM	tblItem a
		JOIN tmpTransferItemValidation b ON b.ItemID = a.ID
		JOIN @StockBalSummary c ON 
			c.ModelID = a.ModelID
			AND c.ColourID = a.ColourID
			AND c.TypeID = a.TypeID
			AND c.CenterID = a.CenterID
WHERE	b.TrnTransferItemValidationID = @TrnTransferItemValidationID

-- lnkTransfer
INSERT INTO lnkTransfer
	(TransferID, StkBalID, ItemID, Active, Remark, UOMID, Qty, SmallestQty, CreateDT, TransferDT, AccessID)
SELECT	a.TransferID, c.StkBalID, b.ItemID, 1, 'Serial range transfer', e.ID, 1, 1, @DateTimeNow, @DateTimeNow, @AccessID
FROM	trnTransferItemValidation a
		JOIN tmpTransferItemValidation b ON b.TrnTransferItemValidationID = a.ID
		JOIN tblItem c ON c.ID = b.ItemID
		JOIN lnkModelUOM d ON c.ModelID = d.ModelID
		JOIN refUOM e ON d.UOMID = e.ID
WHERE	a.ID = @TrnTransferItemValidationID

-- trnTransfer
UPDATE  trnTransfer
SET     StatusID = @StatusApproveID,
        TransferDT = @DateTimeNow,
        LastUpdateDT = @DateTimeNow
WHERE   ID = @TransferID

-- Minus InStock
INSERT INTO tblStkJournal
		(TransType, TransID, StkBalID, Operation, Qty, BeforeQty, AfterQty, CreateDT, AccessID)
SELECT	'Transfer', @TransferID, a.ID, 'M', b.Qty, a.Qty, a.Qty - b.Qty, @DateTimeNow, @AccessID
FROM	tblStkBalance a
		JOIN @StockBalSummary b ON b.InStock_StkBalID = a.ID

UPDATE	a
SET		a.Qty = a.Qty - b.Qty
		, a.LastUpdateDT = @DateTimeNow
		, a.AccessID = @AccessID
FROM	tblStkBalance a
		JOIN @StockBalSummary b ON b.InStock_StkBalID = a.ID

-- Plus Transfer
INSERT INTO tblStkJournal
		(TransType, TransID, StkBalID, Operation, Qty, BeforeQty, AfterQty, CreateDT, AccessID)
SELECT	'Transfer', @TransferID, a.ID, 'P', b.Qty, a.Qty, a.Qty + b.Qty, @DateTimeNow, @AccessID
FROM	tblStkBalance a
		JOIN @StockBalSummary b ON b.Transfer_StkBalID = a.ID

UPDATE	a
SET		a.Qty = a.Qty + b.Qty
		, a.LastUpdateDT = @DateTimeNow
		, a.AccessID = @AccessID
FROM	tblStkBalance a
		JOIN @StockBalSummary b ON b.Transfer_StkBalID = a.ID

-- WH tblStkBalance & tblStkJournal
IF (@WHCenterID > 0)
BEGIN
-- Minus WH InStock
INSERT INTO tblStkJournal
		(TransType, TransID, StkBalID, Operation, Qty, BeforeQty, AfterQty, CreateDT, AccessID)
SELECT	'Transfer', @TransferID, a.ID, 'M', b.Qty, a.Qty, a.Qty - b.Qty, @DateTimeNow, @AccessID
FROM	tblStkBalance a
		JOIN @WHStockBalSummary b ON b.InStock_StkBalID = a.ID

UPDATE	a
SET		a.Qty = a.Qty - b.Qty
		, a.LastUpdateDT = @DateTimeNow
		, a.AccessID = @AccessID
FROM	tblStkBalance a
		JOIN @WHStockBalSummary b ON b.InStock_StkBalID = a.ID
	
-- Plus WH Transfer
INSERT INTO tblStkJournal
		(TransType, TransID, StkBalID, Operation, Qty, BeforeQty, AfterQty, CreateDT, AccessID)
SELECT	'Transfer', @TransferID, a.ID, 'P', b.Qty, a.Qty, a.Qty + b.Qty, @DateTimeNow, @AccessID
FROM	tblStkBalance a
		JOIN @WHStockBalSummary b ON b.Transfer_StkBalID = a.ID

UPDATE	a
SET		a.Qty = a.Qty + b.Qty
		, a.LastUpdateDT = @DateTimeNow
		, a.AccessID = @AccessID
FROM	tblStkBalance a
		JOIN @WHStockBalSummary b ON b.Transfer_StkBalID = a.ID
END

-- Delete temp records after everything finish
DELETE FROM tmpTransferItemValidation WHERE TrnTransferItemValidationID IN (SELECT ID FROM trnTransferItemValidation WHERE TransferID = @TransferID)


COMMIT
END
GO
```
