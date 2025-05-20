---
config:
  theme: neutral
  layout: dagre
  look: neo
---
flowchart TD
    A["Start hotelSearchV2 API Request"] --> B{"Validate Request Body"}
    B -- Valid --> C{"Token Present in Header?"}
    B -- Invalid --> B_ERR["Return 422 Validation Error Response"]
    C -- No --> C_ERR["Return 401 Authentication Failed Response"]
    C -- Yes --> D{"Cache Exists for Search-Key?"}
    D -- No --> D_NEW_START["New Search Flow"]
    D_NEW_START --> D1["Authenticate User"]
    D1 -- Auth Failed --> D1_ERR["Return 401 Authentication Failed Response"]
    D1 -- Auth Success --> D2["Check Client Active Status"]
    D2 -- Client Disabled --> D2_ERR["Return 414 Client Disabled Response"]
    D2 -- Client Active --> D3["Check User Status"]
    D3 -- User Inactive or Blocked --> D3_ERR["Return 401 User Inactive or Blocked Response"]
    D3 -- User Active --> D4["Get Client Configuration"]
    D4 -- Config Not Found --> D4_ERR["Return 400 Client Config Not Found Response"]
    D4 -- Config Found --> D5["Check Hotels Vertical Access"]
    D5 -- No Access --> D5_ERR["Return 400 Hotels Access Not Found Response"]
    D5 -- Has Access --> D6{"Property Search?"}
    D6 -- Yes --> D7["Get Property Giata ID"]
    D6 -- No --> D8["Get Region Details"]
    D7 --> D9["Process Region Details to get Coordinates, Name, etc."]
    D8 --> D9
    D9 --> D10["Get State Info"]
    D10 --> D11["Prepare Initial Cache Data"]
    D11 --> D12["Set Initial Cache in Redis"]
    D12 --> D13_ASYNC_GROUP["Initiate Background Tasks"] & D17["Return 200 Pulling Response"]
    D13_ASYNC_GROUP -.-> D14_TASK["Call searchHotels"] & D15_TASK["Call getFiveStarProperties"] & D16_TASK["Call saveSearch"]
    D17 --> Z["End"]
    D -- Yes --> E["Get Cached Data"]
    E --> F{"Polling Complete in Cache"}
    F -- False (Still Polling) --> F_POLL["Return 200 Pulling Response"]
    F_POLL --> Z
    F -- True (Polling Done) --> G["Results Ready Flow"]
    G --> G1["Get User & ClientConfig from Cached Data"]
    G1 --> G2["Determine Sort Field & Order"]
    G2 --> G3["Create Sort Options"]
    G3 --> G4["Get All Hotels from Session Cache"]
    G4 --> G5_CALL_APPLYSORT["Call applySort with hotels, sortField, sortOrder"]
    G5_CALL_APPLYSORT --> G6_CacheHit{"Property Search & Page Number is 1?"}
    G6_CacheHit -- Yes --> G7_CacheHit["Find Searched Hotel in Sorted List, Prepend if Found with isExactMatch=true"]
    G6_CacheHit -- No --> G8_CacheHit["G8_CacheHit"]
    G7_CacheHit --> G8_CacheHit
    G8_CacheHit --> G9_CacheHit["Get Region from Cached Data"]
    G9_CacheHit --> G10_CacheHit{"Filters Present in Request?"}
    G10_CacheHit -- Yes --> G11_CALL_APPLYFILTERS["Call applyFilters with hotels, req.body.filters"]
    G10_CacheHit -- No --> G12_CacheHit["CacheHit"]
    G11_CALL_APPLYFILTERS --> G12_CacheHit
    G12_CacheHit --> G13_CacheHit["Paginate Filtered or Sorted Hotels"]
    G13_CacheHit --> G14_CacheHit["Fetch Currency Details"]
    G14_CacheHit --> G15_CALL_TRANSFORM["Call tranformSearchResponsewith paginated hotels, user, locationDetails, currency"]
    G15_CALL_TRANSFORM --> G16_CacheHit["Filter Transformed Hotels into Location Results & NearBy Results"]
    G16_CacheHit --> G17_CacheHit["Prepare Final Response Data Object"]
    G17_CacheHit --> G18_CacheHit{"Test Location & API Test Mode Enabled?"}
    G18_CacheHit -- Yes --> G19_CacheHit["Return Static Test Response"]
    G18_CacheHit -- No --> G20_CacheHit["Return 200 Final Search Results with Prepared Data"]
    G19_CacheHit --> Z
    G20_CacheHit --> Z
    F_POLL --> n1["Untitled Node"]
