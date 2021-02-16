# Data Explorer API

The data explorer system provides a automatic extraction & filtering of data in query, rather than requiring users to write their own queries. It is largely accessed via messages on the websocket API, but there is also a REST endpoint.

The data explorer system uses [autoextractor](extractors.md) definitions to determine how data in a given tag should be parsed. If no definition exists for a tag, the extraction generation REST API can be used to create candidate autoextractor definitions; the user should then select and install one of these candidates. Once a definition is in place, clients can request "enriched" entries from the raw search renderer using special commands on the search websocket.

## Data Structures

### Element

Data extracted from entries is represented as an array of Element structures. Each Element represents a single "field" from the data, for instance the source address of a Netflow record. Note that some modules may emit *nested* Elements--see the SubElement field.

* Name: A user-friendly name for the Element.
* Path: A complete path specification for the Element, e.g. `foo.bar.[0]` for the json module.
* Value: The value of whatever was extracted from the data--a string, a number, an IP address, etc.
* SubElements: An (optional) array of Elements which fall "below" this one, for data types which have a natural tree-like structure.
* Filters: A list of potential filters which could be applied to the Element, e.g. "!=", "~", ">".

```
interface Element {
	Name:        string;
	Path:        string;
	Value:       string | boolean | number;
	SubElements: Array<Element> | null;
	Filters:     Array<string> | null;
}
```

### ExploreResult

The ExploreResult type is used to return a set of Elements pulled out of a particular entry. It also includes the name of the module which generated the extraction (e.g. "json") and the string name of the entry's tag (e.g. "syslog").

```
interface ExploreResult {
	Elements:	Array<Element>;
	Module:	string;
	Tag:		string;
}
```


### REST Endpoint Request/Response
The following structures are used for the REST endpoint:

```
interface GenerateAXRequest {
	Tag:     string;
	Entries: Array<SearchEntry>;
}
```

```
interface GenerateAXResponse {
	Extractor:	AXDefinition;
	Entries:	Array<SearchEntry>;
	Explore:	Array<ExploreResult>;
}
```

Refer to the [autoextractors](autoextractors.md) documentation for a description of the AXDefinition type.

### FilterRequest

The FilterRequest type is used to add filters to a search query. An array of Filters can be attached to a ParseSearchRequest or StartSearchRequest object; the filters will be automatically inserted into the query.

* Tag: The tag name we wish to filter (typically taken from an ExploreResult object).
* Module: The module name to filter with (typically taken from an ExploreResult object).
* Path: The path of the element to filter (found in an Element object's Path field).
* Op: An optional filter operation to use (found in an Element object's Filters array).
* Value: An optional value to filter against (found in an Element object's Value field).

Note that if Op and Value are not specified, the "filter" simply extracts the specified Element explicitly, rather than filtering on it.

```
interface FilterRequest {
	Tag:    string;
	Module: string;
	Path:   string;
	Op:     string;
	Value:  string;
}
```

## Extraction Generation REST Endpoint

Before entries can be parsed with the data explorer, they need to have an autoextractor definition installed for their tag. The extraction generation endpoint takes a tag name and one or more entries, and returns a collection of possible extractions. It will return one possible extraction for each data exploration module; the user should select the most appropriate extraction and [install the corresponding autoextractor definition](autoextractors.md).

The endpoint is at `/api/explore/generate`; execute a POST request with the body containing a GenerateAXRequest. The server will respond with a mapping of (string) module names to GenerateAXResponse objects, each representing the extraction generated by that particular module. Each GenerateAXResponse object contains one ExploreResult object in the Explore array per SearchEntry in the Entries array.

The following request contains a single entry:

```
{
  "Tag": "foo",
  "Entries": [
    {
      "TS": "2020-11-02T16:58:56.717034109-07:00",
      "SRC": "",
      "Tag": 0,
      "Data": "ewogICJUUyI6ICIyMDIwLTEwLTE0VDEwOjM1OjQxLjEzNjUyMjg4NC0wNjowMCIsCiAgIlByb3RvIjogInVkcCIsCiAgIkxvY2FsIjogIls6Ol06NTMiLAogICJSZW1vdGUiOiAiNzMuNDIuMTA3LjE4MTo0Nzc0MiIsCiAgIlF1ZXN0aW9uIjogewogICAgIkhkciI6IHsKICAgICAgIk5hbWUiOiAicG9ydGVyLmdyYXZ3ZWxsLmlvLiIsCiAgICAgICJScnR5cGUiOiAxLAogICAgICAiQ2xhc3MiOiAxLAogICAgICAiVHRsIjogNjUsCiAgICAgICJSZGxlbmd0aCI6IDQKICAgIH0sCiAgICAiQSI6ICIyMDguNzEuMTQxLjM0IiwKCSJOb25zZW5zZSI6IFsgMTAwLCAyMDAsIDMwMCBdLAoJIk1vcmVOb25zZW5zZSI6IFsgeyJmb28iOiAiYmFyIn0gXQogIH0KfQ==",
      "Enumerated": null
    }
  ]
}

```

Below is a sample response to the request above. For brevity, only the result from one module ("json") is included, and the Elements array has been shortened.

```
{
	"json": [
		{
			"Extractor": {
				"Name": "foo",
				"Desc": "Auto-generated JSON extraction for tag foo",
				"Module": "json",
				"Params": "TS Proto Local Remote Question",
				"Tag": "foo",
				"Labels": null,
				"UID": 0,
				"GIDs": null,
				"Global": false,
				"UUID": "00000000-0000-0000-0000-000000000000",
				"LastUpdated": "0001-01-01T00:00:00Z"
			},
			"Entries": [
				{
					"TS": "2020-11-02T16:58:56.717034109-07:00",
					"SRC": "",
					"Tag": 0,
					"Data": "ewogICJUUyI6ICIyMDIwLTEwLTE0VDEwOjM1OjQxLjEzNjUyMjg4NC0wNjowMCIsCiAgIlByb3RvIjogInVkcCIsCiAgIkxvY2FsIjogIls6Ol06NTMiLAogICJSZW1vdGUiOiAiNzMuNDIuMTA3LjE4MTo0Nzc0MiIsCiAgIlF1ZXN0aW9uIjogewogICAgIkhkciI6IHsKICAgICAgIk5hbWUiOiAicG9ydGVyLmdyYXZ3ZWxsLmlvLiIsCiAgICAgICJScnR5cGUiOiAxLAogICAgICAiQ2xhc3MiOiAxLAogICAgICAiVHRsIjogNjUsCiAgICAgICJSZGxlbmd0aCI6IDQKICAgIH0sCiAgICAiQSI6ICIyMDguNzEuMTQxLjM0IiwKCSJOb25zZW5zZSI6IFsgMTAwLCAyMDAsIDMwMCBdLAoJIk1vcmVOb25zZW5zZSI6IFsgeyJmb28iOiAiYmFyIn0gXQogIH0KfQ==",
					"Enumerated": null
				}
			],
			"Explore": [
				{
					"Elements": [
						{
							"Name": "TS",
							"Path": "TS",
							"Value": "2020-10-14T10:35:41.136522884-06:00",
							"Filters": [
								"==",
								"!="
							]
						},
						{
							"Name": "Proto",
							"Path": "Proto",
							"Value": "udp",
							"Filters": [
								"==",
								"!="
							]
						},
						{
							"Name": "Local",
							"Path": "Local",
							"Value": "[::]:53",
							"Filters": [
								"==",
								"!="
							]
						},
						{
							"Name": "Remote",
							"Path": "Remote",
							"Value": "73.42.107.181:47742",
							"Filters": [
								"==",
								"!="
							]
						},
						{
							"Name": "Question",
							"Path": "Question",
							"Value": "{\n    \"Hdr\": {\n      \"Name\": \"porter.gravwell.io.\",\n      \"Rrtype\": 1,\n      \"Class\": 1,\n      \"Ttl\": 65,\n      \"Rdlength\": 4\n    },\n    \"A\": \"208.71.141.34\",\n\t\"Nonsense\": [ 100, 200, 300 ],\n\t\"MoreNonsense\": [ {\"foo\": \"bar\"} ]\n  }",
							"SubElements": [
								{
									"Name": "Question.A",
									"Path": "Question.A",
									"Value": "208.71.141.34",
									"Filters": [
										"==",
										"!="
									]
								},

							],
							"Filters": [
								"==",
								"!="
							]
						}
					],
					"Module": "json",
					"Tag": "foo"
				}
			]
		}
	]
}
```

Note the `SubElements` field of the "Question" Element for an example of nesting.

## Search Socket Commands

The `raw` and `text` renderers implement two additional websocket commands, `REQ_GET_EXPLORE_ENTRIES` and `REQ_EXPLORE_TS_RANGE`. These commands mirror the REQ_GET_ENTRIES and REQ_TS_RANGE commands, respectively, but the responses to the data explorer commands will include a field named `Explore`, containing an array of ExploreResult objects (defined above), one per entry.

For example, this command requests the first 10 entries:

```
{
	"ID": 61456,
	"EntryRange": {
		"First": 0,
		"Last": 10
	}
}
```

Here is an example response for a search which returned only one entry:

```
{
  "ID": 61456,
  "EntryCount": 1,
  "AdditionalEntries": false,
  "Finished": true,
  "Entries": [
    {
      "TS": "2020-11-02T16:58:56.717034109-07:00",
      "SRC": "",
      "Tag": 0,
      "Data": "ewogICJUUyI6ICIyMDIwLTEwLTE0VDEwOjM1OjQxLjEzNjUyMjg4NC0wNjowMCIsCiAgIlByb3RvIjogInVkcCIsCiAgIkxvY2FsIjogIls6Ol06NTMiLAogICJSZW1vdGUiOiAiNzMuNDIuMTA3LjE4MTo0Nzc0MiIsCiAgIlF1ZXN0aW9uIjogewogICAgIkhkciI6IHsKICAgICAgIk5hbWUiOiAicG9ydGVyLmdyYXZ3ZWxsLmlvLiIsCiAgICAgICJScnR5cGUiOiAxLAogICAgICAiQ2xhc3MiOiAxLAogICAgICAiVHRsIjogNjUsCiAgICAgICJSZGxlbmd0aCI6IDQKICAgIH0sCiAgICAiQSI6ICIyMDguNzEuMTQxLjM0IiwKCSJOb25zZW5zZSI6IFsgMTAwLCAyMDAsIDMwMCBdLAoJIk1vcmVOb25zZW5zZSI6IFsgeyJmb28iOiAiYmFyIn0gXQogIH0KfQ==",
      "Enumerated": null
    }
  ],
  "Explore": [
    {
      "Elements": [
        {
          "Name": "TS",
          "Path": "TS",
          "Value": "2020-10-14T10:35:41.136522884-06:00",
          "Filters": [
            "==",
            "!="
          ]
        },
        {
          "Name": "Proto",
          "Path": "Proto",
          "Value": "udp",
          "Filters": [
            "==",
            "!="
          ]
        },
        {
          "Name": "Local",
          "Path": "Local",
          "Value": "[::]:53",
          "Filters": [
            "==",
            "!="
          ]
        },
        {
          "Name": "Remote",
          "Path": "Remote",
          "Value": "73.42.107.181:47742",
          "Filters": [
            "==",
            "!="
          ]
        },
        {
          "Name": "Question",
          "Path": "Question",
          "Value": "{\n    \"Hdr\": {\n      \"Name\": \"porter.gravwell.io.\",\n      \"Rrtype\": 1,\n      \"Class\": 1,\n      \"Ttl\": 65,\n      \"Rdlength\": 4\n    },\n    \"A\": \"208.71.141.34\",\n\t\"Nonsense\": [ 100, 200, 300 ],\n\t\"MoreNonsense\": [ {\"foo\": \"bar\"} ]\n  }",
          "SubElements": [
            {
              "Name": "Question.A",
              "Path": "Question.A",
              "Value": "208.71.141.34",
              "Filters": [
                "==",
                "!="
              ]
            }
          ],
          "Filters": [
            "==",
            "!="
          ]
        }
      ],
      "Module": "json",
      "Tag": "foo"
    }
  ]
}

```

## Adding Filters to Queries

The information returned from data explorer renderer commands can be used to construct *filter requests*. Filter requests narrow down the results of a query based on values within the data. For instance, in the example response shown above, we may wish to exclude all entries whose "Proto" field is "udp"; the following FilterRequest implements that filter:

```
{
  "Tag": "foo",
  "Module": "json",
  "Path": "Proto",
  "Op": "!=",
  "Value": "udp"
}
```

Below is a StartSearchRequest which includes the filter:

```
{
  "SearchString": "tag=foo",
  "SearchStart": "2020-01-01T12:01:00.0Z07:00",
  "SearchEnd": "2020-01-01T12:01:30.0Z07:00",
  "Filters": [
    {
      "Tag": "foo",
      "Module": "json",
      "Path": "Proto",
      "Op": "!=",
      "Value": "udp"
    }
  ]
}

```

The server's response will include a re-written SearchString field, suitable for saving in the search library if desired:

```
{
  "SearchString": "tag=foo json Proto!=udp",
  "RawQuery": "tag=foo",
  "RenderModule": "text",
  "RenderCmd": "text",
  "OutputSearchSubproto": "search484",
  "OutputStatsSubproto": "stats484",
  "SearchID": "7947106109",
  "SearchStartRange": "2015-01-01T12:01:00.0Z07:00",
  "SearchEndRange": "2015-01-01T12:01:30.0Z07:00",
  "Background": false
}
```

Filter requests may also be attached to ParseSearchRequest messages to validate the filters.