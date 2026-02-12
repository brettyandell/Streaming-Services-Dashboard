**Streaming Services Dashboard **

Files Included

1. PowerQuery\_AllQueries.md \- All Power Query M code  
2. DAX\_Measures.md \- All DAX measures  
3. StreamingDashboardTheme.json \- Custom dark theme

Requirements :

API Key from TheMovieDB.org

---

**STEP 1: Create a New Power BI Report**

1. Open Power BI Desktop  
2. Save the file as `StreamingDashboard.pbix`

---

**STEP 2: Import the Power Query Queries**  
Create Individual Queries:  
For each query below:

1. Go to Home → Transform Data (opens Power Query Editor)  
2. Home → New Source → Blank Query  
3. Right-click the query → Advanced Editor  
4. Delete all content, paste the query code, click Done  
5. Rename the query (right-click → Rename)

Query 1: Config  
let  
    Config \= \[  
        APIKey \= "API KEY HERE",  
        BaseURL \= "https://api.themoviedb.org/3",  
        WatchRegion \= "US",  
        DaysBack \= 30,  
        MaxPages \= 5  
    \]  
in  
    Config

Query 2: StreamingProviders  
let  
    Providers \= Table.FromRecords({  
        \[Name \= "Netflix", ProviderId \= 8\],  
        \[Name \= "Amazon Prime Video", ProviderId \= 9\],  
        \[Name \= "Disney+", ProviderId \= 337\],  
        \[Name \= "HBO Max", ProviderId \= 384\],  
        \[Name \= "Hulu", ProviderId \= 15\],  
        \[Name \= "Paramount+", ProviderId \= 531\],  
        \[Name \= "Apple TV+", ProviderId \= 350\],  
        \[Name \= "Peacock", ProviderId \= 386\]  
    })  
in  
    Providers

Query 3: fnCallTMDB  
let  
    fnCallTMDB \= (endpoint as text, queryParams as record) as table \=\>  
    let  
        APIKey \= Config\[APIKey\],  
        BaseURL \= Config\[BaseURL\],  
        ParamsWithKey \= Record.AddField(queryParams, "api\_key", APIKey),  
        ParamsList \= Record.FieldNames(ParamsWithKey),  
        QueryString \= Text.Combine(List.Transform(ParamsList, each \_ & "=" & Text.From(Record.Field(ParamsWithKey, \_))), "&"),  
        URL \= BaseURL & endpoint & "?" & QueryString,  
        Source \= try Json.Document(Web.Contents(URL, \[Headers \= \[Accept \= "application/json"\], Timeout \= \#duration(0, 0, 0, 30)\])) otherwise null,  
        Results \= if Source \= null then \#table({"error"}, {{"API call failed"}})  
                  else if Record.HasFields(Source, "results") then Table.FromList(Source\[results\], Splitter.SplitByNothing(), {"Data"})  
                  else \#table({"error"}, {{"No results"}})  
    in  
        Results  
in  
    fnCallTMDB

Query 4: fnGetMovies  
let  
    fnGetMovies \= (providerId as number, providerName as text) as table \=\>  
    let  
        Today \= Date.ToText(Date.From(DateTime.LocalNow()), "yyyy-MM-dd"),  
        StartDate \= Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()), \-Config\[DaysBack\]), "yyyy-MM-dd"),  
        GetPage \= (pageNum as number) as table \=\>  
            let  
                Params \= \[watch\_region \= Config\[WatchRegion\], with\_watch\_providers \= Text.From(providerId),  
                    \#"primary\_release\_date.gte" \= StartDate, \#"primary\_release\_date.lte" \= Today,  
                    sort\_by \= "primary\_release\_date.desc", page \= Text.From(pageNum)\],  
                Results \= fnCallTMDB("/discover/movie", Params)  
            in Results,  
        AllPages \= List.Transform({1..Config\[MaxPages\]}, each try GetPage(\_) otherwise \#table({"Data"}, {})),  
        CombinedPages \= Table.Combine(AllPages),  
        ExpandedData \= if Table.HasColumns(CombinedPages, {"Data"}) then  
            Table.ExpandRecordColumn(CombinedPages, "Data", {"id", "title", "release\_date", "vote\_average", "vote\_count", "overview", "popularity"})  
            else \#table({"id", "title", "release\_date", "vote\_average", "vote\_count", "overview", "popularity"}, {}),  
        AddType \= Table.AddColumn(ExpandedData, "Type", each "Movie", type text),  
        AddService \= Table.AddColumn(AddType, "StreamingService", each providerName, type text),  
        RenamedColumns \= Table.RenameColumns(AddService, {{"id", "TMDBId"}, {"title", "Title"}, {"release\_date", "ReleaseDate"},  
            {"vote\_average", "Rating"}, {"vote\_count", "VoteCount"}, {"overview", "Overview"}, {"popularity", "Popularity"}}),  
        FilteredRows \= Table.SelectRows(RenamedColumns, each \[Title\] \<\> null and \[Title\] \<\> "")  
    in FilteredRows  
in  
    fnGetMovies

Query 5: fnGetTVShows  
let  
    fnGetTVShows \= (providerId as number, providerName as text) as table \=\>  
    let  
        Today \= Date.ToText(Date.From(DateTime.LocalNow()), "yyyy-MM-dd"),  
        StartDate \= Date.ToText(Date.AddDays(Date.From(DateTime.LocalNow()), \-Config\[DaysBack\]), "yyyy-MM-dd"),  
        GetPage \= (pageNum as number) as table \=\>  
            let  
                Params \= \[watch\_region \= Config\[WatchRegion\], with\_watch\_providers \= Text.From(providerId),  
                    \#"first\_air\_date.gte" \= StartDate, \#"first\_air\_date.lte" \= Today,  
                    sort\_by \= "first\_air\_date.desc", page \= Text.From(pageNum)\],  
                Results \= fnCallTMDB("/discover/tv", Params)  
            in Results,  
        AllPages \= List.Transform({1..Config\[MaxPages\]}, each try GetPage(\_) otherwise \#table({"Data"}, {})),  
        CombinedPages \= Table.Combine(AllPages),  
        ExpandedData \= if Table.HasColumns(CombinedPages, {"Data"}) then  
            Table.ExpandRecordColumn(CombinedPages, "Data", {"id", "name", "first\_air\_date", "vote\_average", "vote\_count", "overview", "popularity"})  
            else \#table({"id", "name", "first\_air\_date", "vote\_average", "vote\_count", "overview", "popularity"}, {}),  
        AddType \= Table.AddColumn(ExpandedData, "Type", each "TV Show", type text),  
        AddService \= Table.AddColumn(AddType, "StreamingService", each providerName, type text),  
        RenamedColumns \= Table.RenameColumns(AddService, {{"id", "TMDBId"}, {"name", "Title"}, {"first\_air\_date", "ReleaseDate"},  
            {"vote\_average", "Rating"}, {"vote\_count", "VoteCount"}, {"overview", "Overview"}, {"popularity", "Popularity"}}),  
        FilteredRows \= Table.SelectRows(RenamedColumns, each \[Title\] \<\> null and \[Title\] \<\> "")  
    in FilteredRows  
in  
    fnGetTVShows

Query 6: Movies  
let  
    Providers \= StreamingProviders,  
    MoviesPerProvider \= Table.AddColumn(Providers, "MovieData", each fnGetMovies(\[ProviderId\], \[Name\])),  
    ExpandedMovies \= Table.ExpandTableColumn(MoviesPerProvider, "MovieData",   
        {"TMDBId", "Title", "ReleaseDate", "Rating", "VoteCount", "Overview", "Popularity", "Type", "StreamingService"}),  
    RemovedColumns \= Table.RemoveColumns(ExpandedMovies, {"Name", "ProviderId"}),  
    UniqueMovies \= Table.Distinct(RemovedColumns, {"TMDBId"}),  
    TypedColumns \= Table.TransformColumnTypes(UniqueMovies, {  
        {"TMDBId", Int64.Type}, {"Title", type text}, {"ReleaseDate", type date},  
        {"Rating", type number}, {"VoteCount", Int64.Type}, {"Overview", type text},  
        {"Popularity", type number}, {"Type", type text}, {"StreamingService", type text}}),  
    SortedRows \= Table.Sort(TypedColumns, {{"ReleaseDate", Order.Descending}})  
in  
    SortedRows

Query 7: TVShows  
let  
    Providers \= StreamingProviders,  
    ShowsPerProvider \= Table.AddColumn(Providers, "ShowData", each fnGetTVShows(\[ProviderId\], \[Name\])),  
    ExpandedShows \= Table.ExpandTableColumn(ShowsPerProvider, "ShowData",   
        {"TMDBId", "Title", "ReleaseDate", "Rating", "VoteCount", "Overview", "Popularity", "Type", "StreamingService"}),  
    RemovedColumns \= Table.RemoveColumns(ExpandedShows, {"Name", "ProviderId"}),  
    UniqueShows \= Table.Distinct(RemovedColumns, {"TMDBId"}),  
    TypedColumns \= Table.TransformColumnTypes(UniqueShows, {  
        {"TMDBId", Int64.Type}, {"Title", type text}, {"ReleaseDate", type date},  
        {"Rating", type number}, {"VoteCount", Int64.Type}, {"Overview", type text},  
        {"Popularity", type number}, {"Type", type text}, {"StreamingService", type text}}),  
    SortedRows \= Table.Sort(TypedColumns, {{"ReleaseDate", Order.Descending}})  
in  
    SortedRows

Query 8: StreamingContent  
let  
    CombinedContent \= Table.Combine({Movies, TVShows}),  
    AddRatingCategory \= Table.AddColumn(CombinedContent, "RatingCategory", each   
        if \[Rating\] \>= 8 then "Excellent (8+)" else if \[Rating\] \>= 6 then "Good (6-8)"  
        else if \[Rating\] \>= 4 then "Average (4-6)" else "Below Average (\<4)", type text),  
    AddDaysSinceRelease \= Table.AddColumn(AddRatingCategory, "DaysSinceRelease", each   
        try Duration.Days(DateTime.LocalNow() \- DateTime.From(\[ReleaseDate\])) otherwise null, Int64.Type),  
    AddReleaseWeek \= Table.AddColumn(AddDaysSinceRelease, "ReleaseWeek", each   
        try Date.WeekOfYear(\[ReleaseDate\]) otherwise null, Int64.Type),  
    AddMonthYear \= Table.AddColumn(AddReleaseWeek, "MonthYear", each   
        try Date.ToText(\[ReleaseDate\], "MMM yyyy") otherwise null, type text),  
    AddShortOverview \= Table.AddColumn(AddMonthYear, "ShortOverview", each   
        if \[Overview\] \= null then "No description available"  
        else if Text.Length(\[Overview\]) \> 150 then Text.Start(\[Overview\], 150\) & "..." else \[Overview\], type text),  
    SortedContent \= Table.Sort(AddShortOverview, {{"ReleaseDate", Order.Descending}, {"Popularity", Order.Descending}})  
in  
    SortedContent

Query 9: DateTable  
let  
    DaysBack \= Config\[DaysBack\],  
    StartDate \= Date.AddDays(Date.From(DateTime.LocalNow()), \-DaysBack),  
    EndDate \= Date.From(DateTime.LocalNow()),  
    NumberOfDays \= Duration.Days(EndDate \- StartDate) \+ 1,  
    DateList \= List.Dates(StartDate, NumberOfDays, \#duration(1, 0, 0, 0)),  
    TableFromList \= Table.FromList(DateList, Splitter.SplitByNothing()),  
    RenamedColumns \= Table.RenameColumns(TableFromList, {{"Column1", "Date"}}),  
    ChangedType \= Table.TransformColumnTypes(RenamedColumns, {{"Date", type date}}),  
    AddYear \= Table.AddColumn(ChangedType, "Year", each Date.Year(\[Date\]), Int64.Type),  
    AddMonth \= Table.AddColumn(AddYear, "Month", each Date.Month(\[Date\]), Int64.Type),  
    AddMonthName \= Table.AddColumn(AddMonth, "MonthName", each Date.MonthName(\[Date\]), type text),  
    AddWeekNum \= Table.AddColumn(AddMonthName, "WeekNumber", each Date.WeekOfYear(\[Date\]), Int64.Type),  
    AddDayOfWeek \= Table.AddColumn(AddWeekNum, "DayOfWeek", each Date.DayOfWeekName(\[Date\]), type text)  
in  
    AddDayOfWeek

---

**STEP 3: Configure API Credentials**  
When prompted for credentials:

1. Click Edit Credentials  
2. Select Anonymous  
3. Set Privacy Level to Public  
4. Click Connect

---

**STEP 4: Close & Apply**

1. In Power Query Editor, click Home → Close & Apply  
2. Wait for data to load (may take 1-2 minutes)

---

**STEP 5: Import the Theme**

1. Save the JSON theme code to a file named `StreamingDashboardTheme.json`  
2. In Power BI, go to View → Themes → Browse for themes  
3. Select the JSON file  
4. Click Open

---

**STEP 6: Create the Measures Table**

1. Go to Modeling → New Table  
2. Enter: `Measures = {BLANK()}`  
3. Press Enter

---

**STEP 7: Add DAX Measures**  
For each measure:

1. Click on the Measures table in the Fields pane  
2. Go to Modeling → New Measure  
3. Paste the measure code  
4. Press Enter

---

**STEP 8: Create Relationships**

1. Go to Model view (left sidebar)  
2. Drag StreamingContent\[ReleaseDate\] to DateTable\[Date\]  
3. Ensure it's set to: Many to One, Single direction

---

**Done! 🎉**  
Your dashboard should now be fully functional with live data from TMDB\!
