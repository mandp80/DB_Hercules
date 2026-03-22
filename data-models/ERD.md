## ACS Cash Data

<img width="731" height="569" alt="image" src="https://github.com/user-attachments/assets/4067a550-f122-4686-bbba-f0d8892c4fee" />

-  **Cash:** A transactional table stored Cash flow for IR Derivative realised, purchasing - selling Bonds and Loans business corresponding to Trade.
---
## FDW Trade Data

<img width="731" height="569" alt="image" src="https://github.com/user-attachments/assets/d34187b0-17e5-46b1-8eed-2f9027a17949" />

- **Trade:** A transactional table stored trade information including trade balance,UBR details, trade type, currency and other relevant data. The table sources its data from FDW (CRES)
- **Trade_IFRS:** A transactional table stored trade information including trade Level, product Code and other relevant data. The table sources its data from FDW (CRES)

---
## FX Rates

<img width="553" height="504" alt="image" src="https://github.com/user-attachments/assets/43f8b579-71b7-45c5-9cc5-a468ad69b352" />

---
## Other Important Tables
- **Cash Manual:** A transactional table stored Cash flow for Manual & Kannon Cash. The table sources its data from GFT in .csv files manually. Also users can create manual cash from UI for missing Trades.

- **Levelling:** A transactional table stored level 3 Trade information sources from Kannon received in .xlsx files manually.
- **Opening Balance:** A transactional table stored opening balance respective to UBRs received in .xlsx file.
- **Adjustments:** Use as a history table to captured any changes respective to Trade from UI.
- **Trade Manual & Trade IFRS Manual:** A transactional table stores manually created records that are not found in the FDW feed sourced from UI.
- **Trade Link:** A linkage table stores link values between Trades and Trade_manual sourced from UI.

---

