# Qlik-Internship-Airline-Data-Analysis

###DATASET LINK - **[https://drive.google.com/drive/u/1/search?q=airline%20dataset](https://drive.google.com/file/d/17vdTsRJaBSvoi9GP5vS8VmlFJtpD0NQm/view?usp=sharing)**

**Dataset Loading in Qlik Sense:** https://fxiculp5kb2zxqx.sg.qlikcloud.com/sense/app/4256ce68-3e1a-40e5-91e2-5ab70cfa4661/datamanager/datamanager

**Data Preprocessing:**
**Main File:**
SET ThousandSep=',';
SET DecimalSep='.';
SET MoneyThousandSep=',';
SET MoneyDecimalSep='.';
SET MoneyFormat='$ ###0.00;-$ ###0.00';
SET TimeFormat='h:mm:ss TT';
SET DateFormat='M/D/YYYY';
SET TimestampFormat='M/D/YYYY h:mm:ss[.fff] TT';
SET FirstWeekDay=6;
SET BrokenWeeks=1;
SET ReferenceDay=0;
SET FirstMonthOfYear=1;
SET CollationLocale='en-US';
SET CreateSearchIndexOnReload=1;
SET MonthNames='Jan;Feb;Mar;Apr;May;Jun;Jul;Aug;Sep;Oct;Nov;Dec';
SET LongMonthNames='January;February;March;April;May;June;July;August;September;October;November;December';
SET DayNames='Mon;Tue;Wed;Thu;Fri;Sat;Sun';
SET LongDayNames='Monday;Tuesday;Wednesday;Thursday;Friday;Saturday;Sunday';
SET NumericalAbbreviation='3:k;6:M;9:G;12:T;15:P;18:E;21:Z;24:Y;-3:m;-6:μ;-9:n;-12:p;-15:f;-18:a;-21:z;-24:y';

**Auto-Generated Section**
Set dataManagerTables = '','Cleaned Airline Data';
//This block renames script tables from non generated section which conflict with the names of managed tables

For each name in $(dataManagerTables) 
    Let index = 0;
    Let currentName = name; 
    Let tableNumber = TableNumber(name); 
    Let matches = 0; 
    Do while not IsNull(tableNumber) or (index > 0 and matches > 0)
        index = index + 1; 
        currentName = name & '-' & index; 
        tableNumber = TableNumber(currentName) 
        matches = Match('$(currentName)', $(dataManagerTables));
    Loop 
    If index > 0 then 
            Rename Table '$(name)' to '$(currentName)'; 
    EndIf; 
Next; 
Set dataManagerTables = ;


Unqualify *;

__countryAliasesBase:
LOAD
	Alias AS [__Country],
	ISO3Code AS [__ISO3Code]
FROM [lib://DataFiles/countryAliases.qvd]
(qvd);

__countryGeoBase:
LOAD
	ISO3Code AS [__ISO3Code],
	ISO2Code AS [__ISO2Code],
	Polygon AS [__Polygon]
FROM [lib://DataFiles/countryGeo.qvd]
(qvd);

__countryName2IsoThree:
MAPPING LOAD
	__Country,
	__ISO3Code
RESIDENT __countryAliasesBase;

__countryCodeIsoThree2Polygon:
MAPPING LOAD
	__ISO3Code,
	__Polygon
RESIDENT __countryGeoBase;

__countryCodeIsoTwo2Polygon:
MAPPING LOAD
	__ISO2Code,
	__Polygon
RESIDENT __countryGeoBase;

[Cleaned Airline Data]:
LOAD
	[Passenger ID],
	[First Name],
	[Last Name],
	[Gender],
	[Age],
	[Nationality],
	[Airport Name],
	[Airport Country Code],
	[Country Name],
	[Airport Continent],
	[Continents],
	Date([Departure Date] ) AS [Departure Date],
	[Arrival Airport],
	[Pilot Name],
	[Flight Status],
	APPLYMAP( '__countryCodeIsoThree2Polygon', APPLYMAP( '__countryName2IsoThree', LOWER([Nationality])), '-') AS [Cleaned Airline Data.Nationality_GeoInfo],
	APPLYMAP( '__countryCodeIsoTwo2Polygon', UPPER([Airport Country Code]), '-') AS [Cleaned Airline Data.Airport Country Code_GeoInfo],
	APPLYMAP( '__countryCodeIsoThree2Polygon', APPLYMAP( '__countryName2IsoThree', LOWER([Country Name])), '-') AS [Cleaned Airline Data.Country Name_GeoInfo]
 FROM [lib://DataFiles/Cleaned Airline Data.csv]
(txt, utf8, embedded labels, delimiter is ',', msq);



TAG FIELD [Nationality] WITH '$geoname', '$relates_Cleaned Airline Data.Nationality_GeoInfo';
TAG FIELD [Cleaned Airline Data.Nationality_GeoInfo] WITH '$geopolygon', '$hidden', '$relates_Nationality';
TAG FIELD [Airport Country Code] WITH '$geoname', '$relates_Cleaned Airline Data.Airport Country Code_GeoInfo';
TAG FIELD [Cleaned Airline Data.Airport Country Code_GeoInfo] WITH '$geopolygon', '$hidden', '$relates_Airport Country Code';
TAG FIELD [Country Name] WITH '$geoname', '$relates_Cleaned Airline Data.Country Name_GeoInfo';
TAG FIELD [Cleaned Airline Data.Country Name_GeoInfo] WITH '$geopolygon', '$hidden', '$relates_Country Name';

DROP TABLES __countryAliasesBase, __countryGeoBase;
[autoCalendar]: 
  DECLARE FIELD DEFINITION Tagged ('$date')
FIELDS
  Dual(Year($1), YearStart($1)) AS [Year] Tagged ('$axis', '$year'),
  Dual('Q'&Num(Ceil(Num(Month($1))/3)),Num(Ceil(NUM(Month($1))/3),00)) AS [Quarter] Tagged ('$quarter', '$cyclic'),
  Dual(Year($1)&'-Q'&Num(Ceil(Num(Month($1))/3)),QuarterStart($1)) AS [YearQuarter] Tagged ('$yearquarter', '$qualified'),
  Dual('Q'&Num(Ceil(Num(Month($1))/3)),QuarterStart($1)) AS [_YearQuarter] Tagged ('$yearquarter', '$hidden', '$simplified'),
  Month($1) AS [Month] Tagged ('$month', '$cyclic'),
  Dual(Year($1)&'-'&Month($1), monthstart($1)) AS [YearMonth] Tagged ('$axis', '$yearmonth', '$qualified'),
  Dual(Month($1), monthstart($1)) AS [_YearMonth] Tagged ('$axis', '$yearmonth', '$simplified', '$hidden'),
  Dual('W'&Num(Week($1),00), Num(Week($1),00)) AS [Week] Tagged ('$weeknumber', '$cyclic'),
  Date(Floor($1)) AS [Date] Tagged ('$axis', '$date', '$qualified'),
  Date(Floor($1), 'D') AS [_Date] Tagged ('$axis', '$date', '$hidden', '$simplified'),
  If (DayNumberOfYear($1) <= DayNumberOfYear(Today()), 1, 0) AS [InYTD] ,
  Year(Today())-Year($1) AS [YearsAgo] ,
  If (DayNumberOfQuarter($1) <= DayNumberOfQuarter(Today()),1,0) AS [InQTD] ,
  4*Year(Today())+Ceil(Month(Today())/3)-4*Year($1)-Ceil(Month($1)/3) AS [QuartersAgo] ,
  Ceil(Month(Today())/3)-Ceil(Month($1)/3) AS [QuarterRelNo] ,
  If(Day($1)<=Day(Today()),1,0) AS [InMTD] ,
  12*Year(Today())+Month(Today())-12*Year($1)-Month($1) AS [MonthsAgo] ,
  Month(Today())-Month($1) AS [MonthRelNo] ,
  If(WeekDay($1)<=WeekDay(Today()),1,0) AS [InWTD] ,
  (WeekStart(Today())-WeekStart($1))/7 AS [WeeksAgo] ,
  Week(Today())-Week($1) AS [WeekRelNo] ;

DERIVE FIELDS FROM FIELDS [Departure Date] USING [autoCalendar] ;

[Airline Dataset]:
LOAD
    [Passenger ID],
    [First Name],
    [Last Name],
    [Gender],
    [Age],
    [Nationality],
    [Airport Name],
    [Airport Country Code],
    [Country Name],
    [Airport Continent],
    [Continents],
    [Departure Date],
    [Arrival Airport],
    [Pilot Name],
    [Flight Status]
WHERE NOT ([Arrival Airport] = '0' OR [Arrival Airport] = '-');

