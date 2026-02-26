let
    // API URL
    Source = Json.Document(
        Web.Contents(
            "https://query1.finance.yahoo.com/v8/finance/chart/%5ENSEI",
            [
                Query = [
                    interval = "1d",
                    range = "1mo"
                ]
            ]
        )
    ),

    // Navigate JSON
    Chart = Source[chart],
    Result = Chart[result]{0},
    Meta = Result[meta],
    Timestamps = Result[timestamp],
    Indicators = Result[indicators],
    Quote = Indicators[quote]{0},

    OpenList = Quote[open],
    HighList = Quote[high],
    LowList = Quote[low],
    CloseList = Quote[close],
    VolumeList = Quote[volume],

    // Combine into table
    Combined = List.Zip({Timestamps, OpenList, HighList, LowList, CloseList, VolumeList}),

    TableFromList = Table.FromRows(
        Combined,
        {"Timestamp", "Open", "High", "Low", "Close", "Volume"}
    ),

    // Convert Unix timestamp to DateTime (IST)
    AddDateTime = Table.TransformColumns(
        TableFromList,
        {
            {"Timestamp", each #datetime(1970,1,1,0,0,0) + #duration(0,0,0,_) + #duration(0,5,30,0), type datetime}
        }
    ),

    RenameColumn = Table.RenameColumns(AddDateTime, {{"Timestamp", "DateTime"}}),

    ChangeTypes = Table.TransformColumnTypes(
        RenameColumn,
        {
            {"Open", type number},
            {"High", type number},
            {"Low", type number},
            {"Close", type number},
            {"Volume", Int64.Type}
        }
    ),

    Source1 = ChangeTypes,

    // Ensure sorted
    Sorted = Table.Sort(Source1, {{"DateTime", Order.Ascending}}),

    // Create Year & Week Number
    AddYear = Table.AddColumn(Sorted, "Year", each Date.Year(Date.From([DateTime])), Int64.Type),
    AddWeek = Table.AddColumn(AddYear, "Week", each Date.WeekOfYear(Date.From([DateTime]), Day.Monday), Int64.Type),

    // Group into Weekly OHLC
    WeeklyGrouped = Table.Group(
        AddWeek,
        {"Year", "Week"},
        {
            {"WeekStart", each List.Min([DateTime]), type datetime},
            {"High", each List.Max([High]), type number},
            {"Low", each List.Min([Low]), type number},
            {"Close", each List.Last([Close]), type number}
        }
    ),

    // Sort weeks
    WeeklySorted = Table.Sort(WeeklyGrouped, {{"WeekStart", Order.Ascending}}),

    // Add index
    AddIndex = Table.AddIndexColumn(WeeklySorted, "Index", 0, 1, Int64.Type),

    // Previous Week Values
    AddPrevHigh = Table.AddColumn(AddIndex, "PrevHigh", each try AddIndex[High]{[Index]-1} otherwise null),
    AddPrevLow = Table.AddColumn(AddPrevHigh, "PrevLow", each try AddIndex[Low]{[Index]-1} otherwise null),
    AddPrevClose = Table.AddColumn(AddPrevLow, "PrevClose", each try AddIndex[Close]{[Index]-1} otherwise null),

    // Pivot
    AddPivot = Table.AddColumn(AddPrevClose, "Pivot", each 
        if [PrevHigh] <> null then 
            ([PrevHigh] + [PrevLow] + [PrevClose]) / 3 
        else null, type number),

    // R1
    AddR1 = Table.AddColumn(AddPivot, "R1", each 
        if [Pivot] <> null then (2 * [Pivot]) - [PrevLow] else null, type number),

    // S1
    AddS1 = Table.AddColumn(AddR1, "S1", each 
        if [Pivot] <> null then (2 * [Pivot]) - [PrevHigh] else null, type number),

    // R2
    AddR2 = Table.AddColumn(AddS1, "R2", each 
        if [Pivot] <> null then [Pivot] + ([PrevHigh] - [PrevLow]) else null, type number),

    // S2
    AddS2 = Table.AddColumn(AddR2, "S2", each 
        if [Pivot] <> null then [Pivot] - ([PrevHigh] - [PrevLow]) else null, type number),

    // Clean up
    Final = Table.RemoveColumns(AddS2, {"Index", "PrevHigh", "PrevLow", "PrevClose"})

in
    Final
