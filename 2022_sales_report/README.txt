ABOUT THIS PROJECT

In this project, we're going to emulate a sales report from a company. We have a flat file with all sales data, including data from customers, products, geography data, manufacterer and others, and then we also have another flat file with forecasting data about sales.
Interesting thing is that our flat files are in different levels of granularity, so we'll have to do some work to get valuable insights.

**Main challenge here is to create our data model**

- As we started with a denormalized model, where we have all data from sales in a flat file (CSV), we'll first model
our data into a Star schema:
	* Starting from the entire flat file, we divided into diferent tables, which represent each dimension in our star-schema.
	* We do some data cleansing in the process.
	* We create Customer first name and last name, separating the email column.
	* Remove "()" from email
- Data table has been created through a blank query, using an external M query:

//Create Date Dimension
(StartDate as date, EndDate as date)=>

let
    //Capture the date range from the parameters
    StartDate = #date(Date.Year(StartDate), Date.Month(StartDate), 
    Date.Day(StartDate)),
    EndDate = #date(Date.Year(EndDate), Date.Month(EndDate), 
    Date.Day(EndDate)),

    //Get the number of dates that will be required for the table
    GetDateCount = Duration.Days(EndDate - StartDate),

    //Take the count of dates and turn it into a list of dates
    GetDateList = List.Dates(StartDate, GetDateCount, 
    #duration(1,0,0,0)),

    //Convert the list into a table
    DateListToTable = Table.FromList(GetDateList, 
    Splitter.SplitByNothing(), {"Date"}, null, ExtraValues.Error),

    //Create various date attributes from the date column
    //Add Year Column
    YearNumber = Table.AddColumn(DateListToTable, "Year", 
    each Date.Year([Date])),

    //Add Quarter Column
    QuarterNumber = Table.AddColumn(YearNumber , "Quarter", 
    each "Q" & Number.ToText(Date.QuarterOfYear([Date]))),

    //Add Week Number Column
    WeekNumber= Table.AddColumn(QuarterNumber , "Week Number", 
    each Date.WeekOfYear([Date])),

    //Add Month Number Column
    MonthNumber = Table.AddColumn(WeekNumber, "Month Number", 
    each Date.Month([Date])),

    //Add Month Name Column
    MonthName = Table.AddColumn(MonthNumber , "Month", 
    each Date.ToText([Date],"MMMM")),

    //Add Day of Week Column
    DayOfWeek = Table.AddColumn(MonthName , "Day of Week", 
    each Date.ToText([Date],"dddd"))

in
    DayOfWeek 

- We'll add now the Budget-Forecast dataset, with forecast analysis, but we'll have some troubles in our model. The budget analysis in our new table Forecast is intended in a category level, and our 'Product' table is at a product level, indeed, so we can't create a relationship between them. To fix this, we'll create a whole new dimension 'Category':
	* First, we'll make a copy of Product table, and remove all columns except Category and Segment.
	* Then we'll remove duplicates, and this way we'll have a unique value for every combination between this two columns.
	* Finally, we'll add a unique identifier for each combination through an index column that we named CatSegSK, for SURROGATE KEY.
	* Now we need to add this new SURROGATE KEY into our product table, by merging both tables with a double union key (both category and segment columns).
	* We'll repeat this process with our Forecast table.

- We'll face the last challenge. For this, let's imagine an scenario where there are two types of interest in date column. In one hand, we have sales team, for instance, whom would like to see data taking the order date. But on the other hand, we have logistic team, who would like to see reports based on shipping date. To evaluate this scenario, first we're going to create a 'Shipping date' column, starting from the original 'Date' column, which we'll now name 'Order date'. For the 'shipping date', we'll create it with M lenguage, by simply adding 3 days to order date.
- Now that we have both columns, we'll add a relationship from Date[Shipping Date] to Sales, and we'll see that now there are two lines between this columns, one for each column. However, now our last relationship is inactive.
- So, we recognize the problem: by default, we're able to see only the results for the active relationship. So, how can we get data from both relationships, Order Date and Shipping Date? We'll achieve this by bypassing the active relationship with a new measure using DAX USERELATIONSHIP(), in case we need it.


*Only a small part of the file has been uploaded, due to file size issues.