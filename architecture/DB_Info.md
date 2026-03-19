  <img width="1447" height="564" alt="image" src="https://github.com/user-attachments/assets/5dd2fb2e-eed2-440f-baca-3d70d6199062" />
  <img width="731" height="569" alt="image" src="https://github.com/user-attachments/assets/4c2f067c-89ba-4551-be95-ed6d19e0dced" />
  <img width="731" height="569" alt="image" src="https://github.com/user-attachments/assets/a85e2e7d-fda0-4b2e-b027-f68dd4c6a590" />
  <img width="553" height="504" alt="image" src="https://github.com/user-attachments/assets/825209be-1863-4918-a885-f2847c947034" />

  1	 Trade 	A transactional table stored trade information including trade balance,UBR details, trade type, currency and other relevant data.
The table sources its data from FDW (CRES)
2	Trade_IFRS	A transactional table stored trade information including trade Level, product Code and other relevant data.
The table sources its data from FDW (CRES)
3	CASH	
A transactional table stored Cash flow for IR Derivative realised, purchasing - selling Bonds and Loans business corresponding to Trade.

Until June 2025, the table sources its data from Computron. After this period, it will retrieve information from ACS.

4	CASH_MANUAL	
A transactional table stored Cash flow for Manual & Kannon Cash. The table sources its data from GFT in .csv files manually.

Also users can create manual cash from UI for missing Trades.

5	GVG_Levelling	A transactional table stored level 3 Trade information sources from Kannon received in .xlsx files manually.
6	Imp_BF_Open_Balance	A transactional table stored opening balance respective to UBRs received in .xlsx file.
7	
Adjustments

AdjustmentsNetting

Use as a history table to captured any changes respective to Trade from UI. 
8	
Trade_Manual

Trade_IFRS_Manual

A transactional table stores manually created records that are not found in the FDW feed sourced from UI.
9	Trade_Link	A linkage table stores link values between Trades and Trade_manual sourced from UI.
10	RFT_Y_SUMM	A transactional table store post netted values.
11	
Cash_Source

Cash_Type

GVG_Mapping

GVG_SubType

GVG_Type

Population_Type

Product

RFT_Dates

RFT_DETAIL_YTD_Comments

IFRS_POSITION_AGGR

RFT_Template_Combined

RFT_Template_UBR

RFT_USER

Trade_Type

Cash_Included

Master tables


## ACS Cash Job

2.2. Computron interface Job
The Computron interface application consists of several procedures and processes responsible for migrating and processing data between Computron database as input tables to Cerberus database tables as output.

The application is divided into a main Job with 10 different steps where the procedures are migrated. The ordered steps are these:



### STEP 0: getReportingDates()
This step is always going to be executed, no matter which steps have you chosen because it's necessary to run the application. It gets the cobDate from database. We are using this date as endDate to filter data from Computron input database.



### STEP 1: truncateImpComputron()
This step truncates Imp_Computron table before inserting data to it.

 

### STEP 2: masterComputronToCres()
This step migrates the data from Computron database to Ceberus Imp_Computron table. No big processing is done here, it only makes a mapping from LedgetDTO type to ImpComputron type.

 

### STEP 3: spValidateComputron()
This step checks that Imp_Computron columns class, indexKey, postDtc, journal, batchNum and detlNum are working as primary key, so there are not duplicated records.



### STEP 4: updatesPreSpInsertCash()
This step makes a set of operations before inserting in ads.dbo.Cash table. It truncates ads.dbo.Cash and ads.dbo.Bonds tables, replaces Imp_Computron tref3 column to an empty string for values starting in 11000100 and obtains Computron cash source type from rft.dbo.Cash_Source table.

 

### STEP 5: spInsertCash()
This is a batch processing step where the data is migrated from Imp_Computron table to ads.dbo.Cash table mapping from ImpComputron to CashADS type.

 Before inserting in ads.dbo.Cash table there are several updates in the processor, it updates tref3, triface, cost_center, currency, book_id, book_code, cash_code and tran_code and finally eur_total, currency_id and dt_cash based in some conditions and looking at tables marup.dbo.Trade_Book, ads.dbo.Calendar, ads.dbo.Currency and ads.dbo.Rate.

 

### STEP 6: spInsertBonds()
This step is a tasklet which joins data from Cash with cash type “B” (buy) and cash type “S” (sell) based in some conditions, like the sum of the amounts from a buy and sell record matched must be between -1 and 1.

The result of this join is inserted on ads.dbo.Bonds.

 

 ### STEP 7: updateFinalCashData()
 This step is a taklet which joins ads.dbo.Cash and ads.dbo.Bonds data previously inserted for cash types Buy and Sell. When two records match it sets buy_sell_adjustment from ads.dbo.Cash to 1.

 

 ### STEP 8: spReviewFX()
 This step is a tasklet which checks that no record in ads.dbo.Cash matches with a join with ads.dbo.Calendar and ads.dbo.Rate when Rate.rateId is null. If there is at least one record matching it will throw an Existing Data Exception.

 

 ### STEP 9 :preSpAppendCash()
 This step is a tasklet which checks that there aren’t any records in rft.dbo.Cash with the actual Reporting Date before inserting them. If there is at least one record with that date it will throw an Existing Data Exception.

 

 ### STEP 10: spAppendCash()
 This is a batch processing step which migrates the data from ads.dbo.Cash to rft.dbo.Cash, mapping it from CashADS to CashRFT type.


## CRES Trade Job

Execution Mechanism: - 

     Spring batch is working in below manner

    Grid Size, Chunk Size are defined in the Config map yaml file to create the batch.

    current Grid Size or Thread =10 and Chunk Size = 500

    Like

               Total Record = 100,000

               Partition size (Grid size or Thread) = 10

               Record per Grid (Partition) 100,000/10 = 10,000

                so here we have one Partition (Grid) responsible for 10,000 records



              Chunk Size = 500

              so 500 records will be read/processed/write in one database commit

              Chunk per Partition= 10,000/500

              Total Chunk = 200

              <img width="1265" height="493" alt="image" src="https://github.com/user-attachments/assets/73df2a64-3fcf-4fa8-8b20-a14c21c7975a" />

              The application is divided into a main Job and 12 different steps depending on the work they perform. The steps ordered from 1 to 12 are as follows:

### STEP 1: truncateImpCresImpCresFacAndTrade()
Truncate the tables: imp_Cres, imp_Cres_Fac and Trade

### STEP 2: masterInputToCres()
Migrate the data from InputData (SRC_TRN_DET) to Outputdata (imp_Cres).

### STEP 3: masterInputToCresFac()
Migrate the data from InputDataSrcFacDet (SRC_FAC_DET) to OutputdataImpCresFac (imp_Cres_Fac).

### STEP 4: validateTrade()
Validate and update data from imp_Cres and Trade.

### STEP 5: masterCresToTradeStepWithIndex()
This step is of type Flow, that is, there are several steps inside. The first two delete and create indexes.
initialDeleteIndexInMasterCresToTrade: Deletes indexes in case they were already created.
initialLoadSmallTablesInMemoryCresToTrade: Load small tables in memory to improve execution time.
initialCreationIndexInMasterCresToTrade: Creates table indexes
masterCresToTradeStep: Update and migrate data from ads.dbo.Imp_Cres to ads.dbo.Trade
updateTradeIFRSPrevEnd: Update TradeIFRS table
updateTradeIFRSManualPrevEnd: Update TradeIFRSManual
deleteIndexInMasterCresToTrade: Delete Indexes
deleteSmallTablesInMemoryToTrade: Delete de tables in memory.

### STEP 6: checkCobDtInTradeRFT()
Checks that in rft.dbo.Trade doesn’t exist any record where the cobDate is the same as the new record’s cobDate.

### STEP 7: masterTradeToTradeRFTStep()
Migrate the data from Trade (ADS.dbo.Trade) to TradeRFT (RFT.dbo.Trade).

### STEP 8: masterTradeToRftTradeIfrs()
Migrate the data from Trade (RFT.dbo.Trade) to TradeRFT (RFT.dbo.Trade_ifrs).

### STEP 9: rftNettingEngineStep()
Does several transformations in RFT.dbo.Trade_ifrs table.

### STEP 10: rftGVGEngineStep()


rftGvgEngine1()
Updates tables RFT.dbo.Trade_ifrs (gvg_subtype field) and RFT.dbo.GVG_Levelling (GVG_Type and GVG_SubType fields).

rftGvgEngine2()
Updates tables RFT.dbo.Trade_ifrs (GVG_FVH_Level field) and RFT.dbo.GVG_Levelling (GVG_Type field).

rftGvgEngine3()
Updates table RFT.dbo.Trade_ifrs (alt_t_id, isin_cusip_secid, fvh_level fields)



### STEP 11: createViewsFlowLinks()
Set ‘vwFlowLink’ variable using RFT.dbo.Cash table and then update TrnCdLink and TrnSrcCdLink fields.



### STEP 12: rftEngineStep() 
This step must be run in the same execution with step 11 because it uses stream 'vwFlowLink' as input.

rftEngineBeforeFillTradeTablesStep()
Truncates tables rft.dbo.TradeQ1_MS, rft.dbo.TradeQ2_MSP, rft.dbo.TradeQ3_MS, rft.dbo.RFT_Table_Summ. Updates DtPos1, DtPos2, DtPos3 and cobDate variables.

stepFillTradeTablesTradeQ1orQ2Msp()
Fill table RFT.dbo.TradeQ1_MSP  from RFT.dbo.vw_Trade_MSP.

stepFillTradeTablesTradeQ3Msp()
Fill table RFT.dbo.TradeQ3_MSP from RFT.dbo.vw_Trade_MSP.

rftEngineAfterFillTradeTablesStep()
Set variable ‘rftTradeProcessorData’ (RftFlows, TradeTypes, RftFlowsMutable, CashTypes, CashIncludedTypes, PopulationTypes).

masterJoinRftTradeStepQ3()
Migrate data from RFT.dbo.TradeQ3_Msp to RFT.dbo.RFT_Table_Summ with several transformations in the ItemProcessor.

masterJoinRftTradeStepQ1()
Migrate data from RFT.dbo.TradeQ1_Msp to RFT.dbo.RFT_Table_Summ with several transformations in the ItemProcessor.

rftTableSummPreFinalStep()
Update RFT.dbo.RFT_Table_Summ (IsNetted, Close_Balance, Tin, Product_Code_Close fields).

masterRftTableSummProdStep()
Update RFT.dbo.RFT_Table_Summ (Product_Code, Tin, Open_Balance, Tout fields).

masterRftTableSummFullStep()
More updates in RFT.dbo.RFT_Table_Summ.

masterRftTableSummFinalStep()
Migrate data from RFT.dbo.RFT_Table_Summ to corresponding table (RFT.dbo.RFT_Y_Summ if TABLE_TYPE_YEAR = 1, or RFT.dbo.RFT_Q_Summ if TABLE_TYPE_QUARTER = 1).


