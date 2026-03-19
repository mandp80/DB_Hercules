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



