# SQL Examples
List of examples.


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
