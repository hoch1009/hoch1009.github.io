---
title: "Purchase reporting in Python"
excerpt: "How we got a decent purchase report out of SAP without extra software and without any modifications to the ERP-system itself."
categories:
  - Python
tags:
  - Python
  - numpy
  - pandas
  - Microsoft Excel
  - purchasing
  - reporting
last_modified_at: 2018-01-09
---

Some years ago my supervisor asked me for a report: he wanted to know our purchase prices for every material we bought within the current financial year. We were using SAP and had played with numerous transactions. We did not find what we were looking for and so I had the challenging task to find a way to get all those information we wanted while considering some corporate peculiarities.

The excel I came up with satisfied the requirements of my supervisor and it was a piece of art I was very proud of. We were the only purchasing department that had that kind of  information. Unfortunately the file ended up getting bigger and bigger (in size) and it kept getting inefficient (handling, updating).
My ambition was to find a way where I could throw in the raw data, have it processed and get back what we were used to work with.

I ended up diving deeper into Python, Numpy, Pandas and finally SQlite.
I did some introductory coding (Python) at [Codeacademy](https://www.codecademy.com/learn/learn-python) and also looked into SQL (also at [Codeacademy](https://www.codecademy.com/learn/learn-sql)) which helped me little.
Other ressources worth mentioning: [Chrisalbon](https://chrisalbon.com), [Stackoverflow](https://stackoverflow.com/), [SQLite.org](https://www.sqlite.org), [pandas documentation](http://pandas.pydata.org).

The code:

~~~ python

#!/usr/bin/env python3.5
import pandas as pd
import numpy as np
import csv
import sqlite3 as sqlite
import xlrd

# reading the extracted rawdata/MCE, define two columns as string and turn it into a DataFrame
df_mce3 = pd.read_excel('Rohdaten/MCE3_012013-102016.xlsx', header=0, converters={'Lieferant': str, 'Monat': str})

# reading a table that assigns SAP commodity to company specific commodity group
df_grdl_wg = pd.read_excel('Rohdaten/Grundlagen.xlsx', sheetname="Warengruppenzuordnung", header=0, parse_cols="A,D,F")

# reading a table that contains a number of suppliers we later do not want to include in some calculations
df_grdl_ka = pd.read_excel('Rohdaten/Grundlagen.xlsx', sheetname="Kreditorenausschluss", header=0, converters={'Kreditorrennummer': str})

# reading a table that contains a number of materials we later do not want to include in some calculations
df_grdl_ma = pd.read_excel('Rohdaten/Grundlagen.xlsx', sheetname="Materialausschluss", header=0)

# reading a table that contains additional data which is not provided by the regular extract
df_grdl_ms = pd.read_excel('Rohdaten/Grundlagen.xlsx', sheetname="Materialstammdaten", header=0, parse_cols="A,D,K,L")

# turning the dataframe of suppliers to exclude into a list for further processing
intlf = []
for x in df_grdl_ka['Kreditorennummer']:
    intlf.append(str(x))

# adding an additional dataframe for a quick "0 or 1" process regarding the suppliers which have to be excluded
df_mce3.insert(2, 'LfA', np.where(np.logical_or((pd.isnull(df_mce3['Lieferant']) == True), (df_mce3['Lieferant'].isin(intlf))), '1', '0') )

# insert a dataframe and fill it with the relevant commodity group from the
df_mce3.insert(6, 'COMP_WG', np.where((pd.isnull(df_mce3['Material']) == True), df_mce3['Warengrp'].map(df_grdl_wg.set_index('wg')['comp_top_wg']), df_mce3['Material'].map(df_grdl_ms.set_index('Material')['COMP_WG'])))

# insert a dataframe for the year (format from excel row "Monat": MM.YYYY) to process it faster
df_mce3.insert(9, 'Jahr', df_mce3['Monat'].map(lambda x: str(x)[-4:]))

# create a list of all years in the file to later use it for iteration
set = df_mce3['Jahr'].unique()
jahre = list(set)
jahre.sort()

# new dataframe to save the cost development for everything purchased from external sources/suppliers
df_ka_ext = pd.DataFrame()

# first row ot the dataframe contains all relevant material (have supplier listed and this supplier is not in the to-exclude list)
df_ka_ext['Material'] = df_mce3.loc[(df_mce3['Lieferant'].notnull()) & (df_mce3['LfA'] == '0'), "Material"].unique()

# calculate the development of costs from external sources
# it calculates the invoice values first, then the invoice quantity and then divides the value by the quantity
print("Berechnung Kostenentwicklung extern")
for jahr in jahre:
    rewert = []
    for material in df_ka_ext['Material']:
        rewert.append(df_mce3.loc[(df_mce3['Material'] == material) & (df_mce3['LfA'] == '0') & (df_mce3['Jahr'] == jahr), 'RechBetr.'].sum())
    df_ka_ext['ReWert_' + str(jahr)] = rewert

    remeng = []
    for material in df_ka_ext['Material']:
        remeng.append(df_mce3.loc[(df_mce3['Material'] == material) & (df_mce3['LfA'] == '0') & (df_mce3['Jahr'] == jahr), 'RE-Menge'].sum())
    df_ka_ext['ReMenge_' + str(jahr)] = remeng

    df_ka_ext.loc[:, 'RePreis_' + str(jahr)] = df_ka_ext.loc[:, 'ReWert_' + str(jahr)].div(df_ka_ext['ReMenge_' + str(jahr)], axis=0, fill_value='0')

# adding the material description - similar to "VLOOKUP" in excel
df_ka_ext.insert(1, 'Bezeichnung', df_ka_ext['Material'].map(df_grdl_ms.set_index('Material')['Materialkurztext']))

# adding the company commodity group - also similar to "VLOOKUP"
df_ka_ext.insert(2, 'WG', df_ka_ext['Material'].map(df_grdl_ms.set_index('Material')['COMP_WG']))

# adding the profitcenter
df_ka_ext.insert(3, 'PCtr', df_ka_ext['Material'].map(df_grdl_ms.set_index('Material')['Prctr']))
df_ka_ext['PCtr'] = df_ka_ext['PCtr'].map(lambda x: str(x)[2:4])

# calculating the saving by subtracting the previous price from the current and then multiplying the result with the the current invoice quantity
if len(jahre) > 1:
    x = len(jahre)
    while x > 1:
        df_ka_ext['Saving_' + str(jahre[-(x-1)])] = df_ka_ext['RePreis_' + str(jahre[-(x-1)])].fillna(df_ka_ext['RePreis_' + str(jahre[-x])])
        df_ka_ext['Saving_' + str(jahre[-(x-1)])] = df_ka_ext['Saving_' + str(jahre[-(x-1)])].sub(df_ka_ext['RePreis_' + str(jahre[-x])], axis=0) * df_ka_ext['ReMenge_' + str(jahre[-(x-1)])]
        df_ka_ext.replace(to_replace=[-np.inf, np.inf], value=[np.NaN, np.NaN], inplace=True)
        x = x - 1

# repeating the steps above for material purchased within the company

df_ka_int = pd.DataFrame()
df_ka_int['Material'] = df_mce3.loc[(df_mce3['Lieferant'].notnull()) & (df_mce3['LfA'] == '1'), "Material"].unique()
print("Berechnung Kostenentwicklung intern")
for jahr in jahre:
    rewert = []
    for material in df_ka_int['Material']:
        rewert.append(df_mce3.loc[(df_mce3['Material'] == material) & (df_mce3['LfA'] == '1') & (df_mce3['Jahr'] == jahr), 'RechBetr.'].sum())
    df_ka_int['ReWert_' + str(jahr)] = rewert

    remeng = []
    for material in df_ka_int['Material']:
        remeng.append(df_mce3.loc[(df_mce3['Material'] == material) & (df_mce3['LfA'] == '1') & (df_mce3['Jahr'] == jahr), 'RE-Menge'].sum())
    df_ka_int['ReMenge_' + str(jahr)] = remeng

    df_ka_int.loc[:, 'RePreis_' + str(jahr)] = df_ka_int.loc[:, 'ReWert_' + str(jahr)].div(df_ka_int['ReMenge_' + str(jahr)], axis=0, fill_value='0')


df_ka_int.insert(1, 'Bezeichnung', df_ka_int['Material'].map(df_grdl_ms.set_index('Material')['Materialkurztext']))

df_ka_int.insert(2, 'WG', df_ka_int['Material'].map(df_grdl_ms.set_index('Material')['COMP_WG']))


df_ka_int.insert(3, 'PCtr', df_ka_int['Material'].map(df_grdl_ms.set_index('Material')['Prctr']))
df_ka_int['PCtr'] = df_ka_int['PCtr'].map(lambda x: str(x)[2:4])

# calculating the saving by subtracting the previous price from the current and then multiplying the result with the the current invoice quantity
if len(jahre) > 1:
    x = len(jahre)
    while x > 1:
        df_ka_int['Saving_' + str(jahre[-(x-1)])] = df_ka_int['RePreis_' + str(jahre[-(x-1)])].fillna(df_ka_int['RePreis_' + str(jahre[-x])])
        df_ka_int['Saving_' + str(jahre[-(x-1)])] = df_ka_int['Saving_' + str(jahre[-(x-1)])].sub(df_ka_int['RePreis_' + str(jahre[-x])], axis=0) * df_ka_int['ReMenge_' + str(jahre[-(x-1)])]
        df_ka_int.replace(to_replace=[-np.inf, np.inf], value=[np.NaN, np.NaN], inplace=True)
        x = x - 1


# TODO:
# KPI
# calculate key performance inidications and save them to a separate dataframe for easy access

# save the dataframes (as well as the original data) as individual tables in a sqlite database
conn = sqlite.connect("database.db")
df_mce3.to_sql("MCE3", conn, if_exists="replace")
df_ka_ext.to_sql("Kostenanalyse_extern", conn, if_exists="replace")
df_ka_int.to_sql("Kostenanalyse_intern", conn, if_exists="replace")
conn.commit()
conn.close()

~~~

What we get:
- materials which have been purchased get two prices (external and internal)
- the relevant commodity groups are assigned to the materials (we could easily filter)
- transaction data and all calculations are saved for detailed inspection if necessary

Now I do not spend hours waiting for excel to finish processing the formulas. I keep the basic data up to date, provide a current extract of the transaction data and start the script. I can work in the meantime what I could not when excel was working.


I translated the comments in the code. Futher translations:

English | German
--- | ---
Warengruppenzuordnung | commodity group assignment
Kreditorenausschluss | supplier exclusion
Kreditorrennummer | supplier code number
Materialausschluss | material exclusion
Materialstammdaten | material master data
Rohdaten | raw data
Grundlagen | basics
Lieferant | supplier
Material  | material
Jahr | year
Monat | month
Berechnung Kostenentwicklung extern | external cost development calculation
Berechnung Kostenentwicklung intern | internal cost development calculation
RechBetr | invoice value
RE-Menge | invoice quantity
RePreis_ | invoice price
Bezeichnung | description
Materialkurztext | material description
Kreditorennummer | supplier code number
