## Work progress
Degiro needs a special implementation.

### What is missing
For the final development of a specific implementation, the developer lacks CSV documents from various Degiro customers. Otherwise, a well-functioning import is not possible.

## PDF documents
PDF documents are not supported because they are obviously not different from CSV documents.

## CSV documents
It certainly needs two CSV documents for complete tracking of all transactions. A link between the individual table rows can be done across documents on the **Order ID** table column.
Account transactions
### Account transactions
Apparently Degiro manages the balance through a money market fund, which charges interest.  
```
Datum,Uhrze,Valutadatum,Produkt,ISIN,Beschreibung,FX,Änderung,,Saldo,,Order-ID
10-02-2021,00:00,09-02-2021,FUNDSHARE UCITS CHF CASH FUND,NL0011280581,Geldmarktfonds Preisänderung (CHF),,CHF,-0.02,CHF,154.30,
```

### Securities transactions
Obviously there is a CSV document with buy and sell transactions. In this one the **dividends** are missing. The following is an excerpt from the transaction document:
```
Datum,Uhrzeit,Produkt,ISIN,Börse,Venue,Anzahl,Kurs,,Wert in Lokalwährung,,Wert,,Wechselkurs,Gebühr,,Gesamt,,Order-ID
29-12-2020,20:13,ALIBABA GROUP HOLDING,US01609W1027,NSY,CDED,4,237.6900,USD,-950.76,USD,-841.21,CHF,1.1291,-0.55,CHF,-841.76,CHF,b7f72d99-6980-426d-9467-0cf8c9722f73
```
Not all columns relevant for import have a designation in the header. These must be entered by the user.
```
Datum;Uhrzeit;Produkt;ISIN;Börse;Venue;Anzahl;Kurs;cin;Wert in Lokalwährung;;Wert;;Wechselkurs;Gebühr;cct;Gesamt;cac;Order-ID
29.12.2020;20:13;ALIBABA GROUP HOLDING;US01609W1027;NSY;CDED;4;237.69;USD;-950.76;USD;-841.21;CHF;1.1291;-0.55;CHF;-841.76;CHF;b7f72d99-6980-426d-9467-0cf8c9722f73
