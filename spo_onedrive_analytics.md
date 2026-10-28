# SharePoint and OneDrive analytics

## A glimpse at tenant activities

```kusto
CloudAppEvents
| where Timestamp >= ago(7d)
| where Application in (
    "Microsoft SharePoint Online", 
    "Microsoft OneDrive for Business"
    )
| summarize Events = count() by ActionType
| sort by Events desc
```

### Inventory of all audit event types

```kusto
CloudAppEvents
| summarize ActionTypes = make_set(ActionType) by Application
```

## What does the daily activity look like?

```kusto
let ViewActions = dynamic([
    "FileAccessed",
    "FilePreviewed"
    ]);
let ShareActions = dynamic([
    "CompanyLinkCreated",
    "CompanyLinkUpdated",
    "SharingLinkCreated",
    "SharingLinkUpdated",
    "AnonymousLinkCreated",
    "AnonymousLinkUpdated",
    "SecureLinkCreated",
    "SecureLinkUpdated",
    "SharingSet",
    "AddedToSharingLink",
    "AddedToSecureLink"
    ]);
CloudAppEvents
| where Timestamp >= ago(30d)
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business"
    )
| extend Category = case(
    ActionType in (ViewActions), "Viewed",
    ActionType == "FileDownloaded", "Downloaded",
    ActionType == "FileUploaded", "Uploaded",
    ActionType == "FileModified", "Edited",
    ActionType in (ShareActions), "Shared",
    "")
| where isnotempty(Category)
| summarize Events = count() by Day = bin(Timestamp, 1d), Category
| sort by Day asc
| render areachart with (kind=stacked)
```

## How is content being shared?

```kusto
CloudAppEvents
| where Timestamp >= ago(30d)
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business"
    )
| where ActionType in (
    "AnonymousLinkCreated",
    "CompanyLinkCreated",
    "SecureLinkCreated",
    "SharingLinkCreated"
    )
| summarize Links = count() by LinkType = replace_string(ActionType, "LinkCreated", "")
| sort by Links desc
| render piechart
```

## What kinds of files do we have?

```kusto
CloudAppEvents
| where Timestamp >= ago(30d)
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business"
    )
| where ActionType in (
    "FileAccessed",
    "FileUploaded",
    "FileModified",
    "FileDownloaded"
    )
| extend FileType = tolower(tostring(RawEventData.SourceFileExtension))
| where isnotempty(FileType)
| summarize Files = dcount(tostring(RawEventData.SourceFileName)) by FileType
| sort by Files desc
| take 8
| render piechart
```

## Who did what?

```kusto
let People = IdentityInfo
    | where Timestamp >= ago(30d)
    | summarize arg_max(Timestamp, *) by AccountObjectId
    | project 
        AccountId = AccountObjectId, 
        Department, 
        JobTitle;
CloudAppEvents
| where Timestamp >= ago(7d)
| where Application == "Microsoft SharePoint Online"
| extend 
    UPN = tostring(RawEventData.UserId), 
    FileName = tostring(RawEventData.SourceFileName)
| summarize Events = count(), LastSeen = max(Timestamp) by 
    AccountId, 
    AccountDisplayName, 
    UPN, 
    ActionType, 
    FileName
| join kind=leftouter People on AccountId
| project 
    AccountDisplayName, 
    UPN,
    AccountId,
    // Department, 
    // JobTitle, 
    ActionType, 
    FileName, 
    Events, 
    LastSeen
| sort by LastSeen desc
| take 50
```

## Who are the most active people?

```kusto
let People = IdentityInfo
    | where Timestamp >= ago(30d)
    | summarize arg_max(Timestamp, *) by AccountObjectId
    | project
        AccountId = AccountObjectId,
        Department,
        JobTitle;
let ViewActions = dynamic([
    "FileAccessed",
    "FilePreviewed"
    ]);
let ShareActions = dynamic([
    "CompanyLinkCreated",
    "CompanyLinkUpdated",
    "SharingLinkCreated",
    "SharingLinkUpdated",
    "AnonymousLinkCreated",
    "AnonymousLinkUpdated",
    "SecureLinkCreated",
    "SecureLinkUpdated",
    "SharingSet",
    "AddedToSharingLink",
    "AddedToSecureLink"
    ]);
CloudAppEvents
| where Timestamp >= ago(30d)
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business"
    )
| where tolong(RawEventData.UserType) == 0
| where isnotempty(AccountDisplayName)
| summarize
    Viewed = countif(ActionType in (ViewActions)),
    Downloaded = countif(ActionType == "FileDownloaded"),
    Uploaded = countif(ActionType == "FileUploaded"),
    Edited = countif(ActionType == "FileModified"),
    Shared = countif(ActionType in (ShareActions)),
    ActiveDays = dcount(bin(Timestamp, 1d))
    by AccountId, AccountDisplayName
| extend TotalActivity = Viewed + Downloaded + Uploaded + Edited + Shared
| join kind=leftouter People on AccountId
| project
    AccountDisplayName,
    // Department,
    // JobTitle,
    Viewed,
    Downloaded,
    Uploaded,
    Edited,
    Shared,
    ActiveDays,
    TotalActivity
| sort by TotalActivity desc
| take 20
```

## What is each site used for?

```kusto
let ShareActions = dynamic([
    "CompanyLinkCreated",
    "CompanyLinkUpdated",
    "SharingLinkCreated",
    "SharingLinkUpdated",
    "AnonymousLinkCreated",
    "AnonymousLinkUpdated",
    "SecureLinkCreated",
    "SecureLinkUpdated",
    "SharingSet",
    "AddedToSharingLink",
    "AddedToSecureLink"
    ]);
CloudAppEvents
| where Timestamp >= ago(30d)
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business"
    )
| extend Site = tostring(split(tostring(RawEventData.SiteUrl), "/")[4])
| where isnotempty(Site)
| summarize
    FilesUploaded = countif(ActionType == "FileUploaded"),
    FoldersCreated = countif(ActionType == "FolderCreated"),
    Shares = countif(ActionType in (ShareActions)),
    Total = count()
    by Site
| sort by Total desc
| take 8
| project
    Site,
    FilesUploaded,
    FoldersCreated,
    Shares
| render columnchart
```

## Which libraries are the busiest?

```kusto
CloudAppEvents
| where Timestamp >= ago(90d)
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business",
    "Microsoft Teams"
    )
| where tolong(RawEventData.ListServerTemplate) == 101
| extend
    SiteUrl = tostring(RawEventData.SiteUrl),
    ListId = tostring(RawEventData.ListId)
| extend
    FromObject = tostring(split(url_decode(substring(ObjectName, strlen(SiteUrl))), "/")[0]),
    FromSource = tostring(split(tostring(RawEventData.SourceRelativeUrl), "/")[0])
| extend Library = iff(isnotempty(FromObject), FromObject, FromSource)
| where isnotempty(Library)
| summarize Events = count(), Users = dcount(AccountId) by SiteUrl, Library, ListId
| sort by Events desc
| take 5
```

## What's happening in this specific document library?

```kusto
let LibraryUrl = "https://contoso.sharepoint.com/sites/LogicAppStorage/LogicAppUploads/";
let LibraryId = "3f2b8c1e-5a47-4d90-9b6e-7c1d2a8e4f05";
CloudAppEvents
| where Timestamp >= ago(90d)
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business"
    )
| where ObjectName startswith LibraryUrl or tostring(RawEventData.ListId) == LibraryId
| where ActionType startswith "File"
| where ActionType !endswith "Extended"
| where ActionType != "FileUploadedPartial"
| where tostring(RawEventData.Platform) != "Service"
| where tolong(RawEventData.UserType) == 0
| where isnotempty(AccountDisplayName)
| extend Activity = case(
    ActionType == "FileAccessed", "Opened",
    ActionType == "FilePreviewed", "Previewed",
    ActionType == "FileDownloaded", "Downloaded",
    ActionType == "FileModified", "Edited",
    ActionType == "FileUploaded", "Uploaded",
    ActionType)
| summarize
    Times = count(),
    Files = dcount(tostring(RawEventData.SourceFileName)),
    FirstSeen = min(Timestamp),
    LastSeen = max(Timestamp)
    by AccountDisplayName, Activity
| sort by Times desc
```

## Which files are opened most, and are people coming back?

```kusto
let Opens = CloudAppEvents
    | where Timestamp >= ago(30d)
    | where Application in (
        "Microsoft SharePoint Online",
        "Microsoft OneDrive for Business"
        )
    | where ActionType in (
        "FileAccessed",
        "FilePreviewed",
        "FileDownloaded"
        )
    | where tostring(RawEventData.Platform) != "Service"
    | where tolong(RawEventData.UserType) == 0
    | where isnotempty(AccountDisplayName)
    | extend
        FileName = tostring(RawEventData.SourceFileName),
        FileType = tostring(RawEventData.SourceFileExtension)
    // | where FileType !in (
    //     "aspx",
    //     ""
    //     )
    // | where FileName !startswith "__"
    // | where FileName !endswith "_SThumb.jpg"
    ;
Opens
| summarize Opens = count() by FileName, AccountDisplayName
| summarize
    TotalOpens = sum(Opens),
    UniqueUsers = dcount(AccountDisplayName),
    RepeatUsers = countif(Opens > 1)
    by FileName
| sort by TotalOpens desc
| take 20
```

## Who opened each file?

```kusto
let Opens = CloudAppEvents
    | where Timestamp >= ago(30d)
    | where Application in (
        "Microsoft SharePoint Online",
        "Microsoft OneDrive for Business"
        )
    | where ActionType in (
        "FileAccessed",
        "FilePreviewed",
        "FileDownloaded"
        )
    | where tostring(RawEventData.Platform) != "Service"
    | where tolong(RawEventData.UserType) == 0
    | where isnotempty(AccountDisplayName)
    | extend
        FileName = tostring(RawEventData.SourceFileName),
        FileType = tostring(RawEventData.SourceFileExtension)
    // | where FileType !in (
    //     "aspx",
    //     ""
    //     )
    // | where FileName !startswith "__"
    // | where FileName !endswith "_SThumb.jpg"
    ;
Opens
| summarize Opens = count() by FileName, AccountDisplayName
| summarize
    TotalOpens = sum(Opens),
    Viewers = make_set(strcat(AccountDisplayName, " (", Opens, ")"))
    by FileName
| sort by TotalOpens desc
| take 20
```

## What did we upload that nobody has opened?

```kusto
let Uploaded = CloudAppEvents
    | where Timestamp >= ago(30d)
    | where Application in (
        "Microsoft SharePoint Online",
        "Microsoft OneDrive for Business"
        )
    | where ActionType == "FileUploaded"
    | extend
        FileName = tostring(RawEventData.SourceFileName),
        Folder = tostring(split(tostring(RawEventData.SourceRelativeUrl), "/")[0]),
        SiteUrl = tostring(RawEventData.SiteUrl)
    | summarize UploadedAt = min(Timestamp) by FileName, Folder, SiteUrl;
let Opened = CloudAppEvents
    | where Timestamp >= ago(30d)
    | where Application in (
        "Microsoft SharePoint Online",
        "Microsoft OneDrive for Business"
        )
    | where ActionType in (
        "FileAccessed",
        "FilePreviewed",
        "FileDownloaded"
        )
    | where tostring(RawEventData.Platform) != "Service"
    | distinct
        FileName = tostring(RawEventData.SourceFileName),
        SiteUrl = tostring(RawEventData.SiteUrl);
Uploaded
| join kind=leftanti Opened on FileName, SiteUrl
| summarize
    NeverOpened = count(),
    OldestUpload = min(UploadedAt)
    by SiteUrl, Folder
| sort by NeverOpened desc
| take 20
```

## Which pages get visits, and how many visitors are unique?

```kusto
CloudAppEvents
// | where Timestamp >= ago(30d)
| where Application in (
    "Microsoft SharePoint Online", 
    "Microsoft OneDrive for Business"
    )
| where ActionType == "PageViewed"
| extend Page = tostring(RawEventData.ObjectId)
| summarize
    TotalVisits = count(),
    VisitsToday = countif(Timestamp >= startofday(now())),
    UniqueVisitors = dcount(AccountId)
    by Page
| sort by TotalVisits desc
| take 25
```

```kusto
CloudAppEvents
// | where Timestamp >= ago(30d)
| where Application in (
    "Microsoft SharePoint Online", 
    "Microsoft OneDrive for Business"
    )
| where ActionType == "PageViewed"
| project RawEventData
| sample 5
```

## How do people get to the content?

```kusto
CloudAppEvents
| where Timestamp >= ago(30d)
| where Application in (
    "Microsoft SharePoint Online",
    "Microsoft OneDrive for Business"
    )
| where ActionType in (
    "FileAccessed",
    "FilePreviewed",
    "FileDownloaded",
    "PageViewed"
    )
| where tostring(RawEventData.Platform) != "Service"
| extend App = tostring(RawEventData.ApplicationDisplayName)
| extend EntryPoint = case(
    App in (
        "Browser",
        "OneDrive Web App (modern)",
        "ClientSidePage",
        "Files"
        ), "Browser or SharePoint page",
    App in (
        "word",
        "excel",
        "powerpoint",
        "WebWord",
        "WebExcel",
        "WebPowerPoint"
        ), "Office app",
    App has "Teams", "Teams",
    App has "OneDrive", "OneDrive sync client",
    "Other")
| summarize
    Events = count(),
    Users = dcount(AccountId)
    by EntryPoint, Device = DeviceType
| sort by Events desc
```
