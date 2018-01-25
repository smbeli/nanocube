# API (v4+)

We will show the query API for Nanocubes using the crime50k example.
In order to create such a nanocube and serve it via http, we
run the following script

```shell
#
# (0) go the the $NANOCUBE/data folder
#
# (1) create the nanocube index file 'crime50k.nanocube'
#
nanocube create <(zcat crime50k.csv.gz) crime50k.map crime50k.nanocube
#
# (2) serve the index via http on port 51234 using the alias 'crimes'
#
nanocube serve 51234 crimes=crime50k.nanocube
```

## schema

To serve the schemas of all the indices being served in port 51234 of
localhost we use:
```url
http://localhost:51234/schema()
```
which gives as a result a `json` array with one entry: the schema of
the crimes index:
```json
[
	{
		"type":"schema",
		"name":"crimes",
		"index_dimensions":[
			{
				"index":0,
				"name":"location",
				"bits_per_level":2,
				"num_levels":25,
				"hint":"spatial",
				"aliases":{
				}
			},
			{
				"index":1,
				"name":"type",
				"bits_per_level":8,
				"num_levels":1,
				"hint":"categorical",
				"aliases":{
					"11":"CRIMINAL TRESPASS",
					"8":"ROBBERY",
					"24":"NON-CRIMINAL",
					"27":"OBSCENITY",
					"30":"NON-CRIMINAL (SUBJECT SPECIFIED)",
					"3":"BATTERY",
					"26":"GAMBLING",
					"23":"INTIMIDATION",
					"2":"THEFT",
					"4":"BURGLARY",
					"17":"LIQUOR LAW VIOLATION",
					"29":"CONCEALED CARRY LICENSE VIOLATION",
					"21":"PROSTITUTION",
					"16":"CRIM SEXUAL ASSAULT",
					"12":"WEAPONS VIOLATION",
					"20":"SEX OFFENSE",
					"7":"DECEPTIVE PRACTICE",
					"10":"OTHER OFFENSE",
					"25":"PUBLIC INDECENCY",
					"14":"PUBLIC PEACE VIOLATION",
					"15":"INTERFERENCE WITH PUBLIC OFFICER",
					"22":"ARSON",
					"1":"CRIMINAL DAMAGE",
					"13":"KIDNAPPING",
					"5":"MOTOR VEHICLE THEFT",
					"0":"NARCOTICS",
					"18":"HOMICIDE",
					"9":"ASSAULT",
					"28":"OTHER NARCOTIC VIOLATION",
					"6":"OFFENSE INVOLVING CHILDREN",
					"19":"STALKING"
				}
			},
			{
				"index":2,
				"name":"time",
				"bits_per_level":1,
				"num_levels":16,
				"hint":"temporal|2013-12-01T06:00:00Z_3600s",
				"aliases":{
				}
			}
		],
		"measure_dimensions":[
			{
				"order":1,
				"name":"count",
				"type":"u32"
			}
		]
	}
]
```

## total

In order to get the total number of crimes indexed in `crimes` we
send the query `q(crimes)`:
```url
http://localhost:51234/q(crimes)
```
The total is 50k, since we indexed 50k records and the `value` measure
we defined in `crime50k.map` is simply a count of records.
```json
[
	{
		"type":"table",
		"numrows":1,
		"index_columns":[
		],
		"measure_columns":[
			{
				"name":"count",
				"values":[
					50000.000000
				]
			}
		]
	}
]
```

## location

The quadtree convention we use is the following:
```
0 bottom-left  quadrant (in the first split of the mercator projection contains South-America)
1 bottom-right quadrant (in the first split of the mercator projection contains Africa and South Asia)
2 top-left     quadrant (in the first split of the mercator projection contains North America)
3 top-right    quadrant (in the first split of the mercator projection contains Europe and North Asia)
```
If we want to count the number of crimes in a 256x256 quadtree-aligned grid
of the sub-cell (top-left, bottom-right, top-left) or (2,1,2), we run the query
below. Note that the cell (2,1,2) contains Chicago.
```
http://localhost:29512/q(crimes.b('location',dive(p(2,1,2),8)))
```
The result is 
```json
[
	{
		"type":"table",
		"numrows":8,
		"index_columns":[
			{
				"name":"location",
				"hint":"none",
				"values_per_row":11,
				"values":[
					2,1,2,0,0,0,0,1,2,3,3,2,1,2,0,0,0,0,1,3,0,2,2,1,2,0,0,0,0,1,3,0,3,2,1,2,0,0,0,0,1,3,1,2,2,1,2,0,0,0,0,1,3,2,0,2,1,2,0,0,0,0,1,3,2,1,2,1,2,0,0,0,0,1,3,2,2,2,1,2,0,0,0,0,1,3,2,3
				]
			}
		],
		"measure_columns":[
			{
				"name":"count",
				"values":[
					272.000000,416.000000,12268.000000,224.000000,6838.000000,16913.000000,5460.000000,7609.000000
				]
			}
		]
	}
]
```

Note that the way the resulting grid cells are described are as paths. Since we
subdivided the cell (2,1,2) at depth 3 more 8 times (the depth of the dive), we
get that each grid cell is described by a path of 11 subdivisions. So the way
to read the result above is to split location 'values' array at every 11-th entry.
The final correspondence can also be better seen by generating the above resul in
text format.
```
http://localhost:29512/format('text');q(crimes.b('location',dive(p(2,1,2),8)))
```
which yields the following text
```
                                          location                           count
                             2,1,2,0,0,0,0,1,2,3,3                      272.000000
                             2,1,2,0,0,0,0,1,3,0,2                      416.000000
                             2,1,2,0,0,0,0,1,3,0,3                    12268.000000
                             2,1,2,0,0,0,0,1,3,1,2                      224.000000
                             2,1,2,0,0,0,0,1,3,2,0                     6838.000000
                             2,1,2,0,0,0,0,1,3,2,1                    16913.000000
                             2,1,2,0,0,0,0,1,3,2,2                     5460.000000
                             2,1,2,0,0,0,0,1,3,2,3                     7609.000000
```
If instead of the path we just want the local (x,y) coordinate of the 256x256 grid,
we can use the hint `'img8'` as the last parameter of the binding `.b(...)`.
```
http://localhost:29512/format('text');q(crimes.b('location',dive(p(2,1,2),8),'img8'))
```
We now get coordinates `x` and `y` both in {0,1,...,255}
```
                                          location                           count
                                              11,7                      272.000000
                                              12,5                      416.000000
                                              13,5                    12268.000000
                                              14,5                      224.000000
                                              12,6                     6838.000000
                                              13,6                    16913.000000
                                              12,7                     5460.000000
                                              13,7                     7609.000000
```
if we want the global coord of the grid we generated we can use the hint `'img11'`:
```
http://localhost:29512/format('text');q(crimes.b('location',dive(p(2,1,2),8),'img11'))
```
We now get coordinates `x` and `y` both in {0,1,...,2047}. Note that the `y`
coordinate of the cells grow bottom-up, Chicago is above the Equator and
{1285,1286,1287} are in the upper half of 2048 or 2^11.

```
                                          location                           count
                                          523,1287                      272.000000
                                          524,1285                      416.000000
                                          525,1285                    12268.000000
                                          526,1285                      224.000000
                                          524,1286                     6838.000000
                                          525,1286                    16913.000000
                                          524,1287                     5460.000000
                                          525,1287                     7609.000000
```

## type

If we want to count the number of crimes by `type` of crime, we run the query
below for either numeric or textual categories.
```url
# numerical
http://localhost:51234/q(crimes.b('type',dive(1)))
```
yields the following json
```json
[
	{
		"type":"table",
		"numrows":31,
		"index_columns":[
			{
				"name":"type",
				"hint":"none",
				"values_per_row":1,
				"values":[
					0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26,27,28,29,30
				]
			}
		],
		"measure_columns":[
			{
				"name":"count",
				"values":[
					5742.000000,4660.000000,11367.000000,8990.000000,2933.000000,2226.000000,456.000000,2190.000000,2132.000000,2629.000000,3278.000000,1429.000000,489.000000,46.000000,441.000000,229.000000,181.000000,69.000000,69.000000,20.000000,119.000000,211.000000,63.000000,21.000000,1.000000,2.000000,2.000000,1.000000,2.000000,1.000000,1.000000
				]
			}
		]
	}
]
```
using the hint `'name'` we get path aliases (*ie* the category names)
```url
# numerical
http://localhost:51234/q(crimes.b('type',dive(1)),'name')
```
results in
```json
[
	{
		"type":"table",
		"numrows":31,
		"index_columns":[
			{
				"name":"type",
				"hint":"name",
				"values_per_row":1,
				"values":[
					"NARCOTICS","CRIMINAL DAMAGE","THEFT","BATTERY","BURGLARY","MOTOR VEHICLE THEFT","OFFENSE INVOLVING CHILDREN","DECEPTIVE PRACTICE","ROBBERY","ASSAULT","OTHER OFFENSE","CRIMINAL TRESPASS","WEAPONS VIOLATION","KIDNAPPING","PUBLIC PEACE VIOLATION","INTERFERENCE WITH PUBLIC OFFICER","CRIM SEXUAL ASSAULT","LIQUOR LAW VIOLATION","HOMICIDE","STALKING","SEX OFFENSE","PROSTITUTION","ARSON","INTIMIDATION","NON-CRIMINAL","PUBLIC INDECENCY","GAMBLING","OBSCENITY","OTHER NARCOTIC VIOLATION","CONCEALED CARRY LICENSE VIOLATION","NON-CRIMINAL (SUBJECT SPECIFIED)"
				]
			}
		],
		"measure_columns":[
			{
				"name":"count",
				"values":[
					5742.000000,4660.000000,11367.000000,8990.000000,2933.000000,2226.000000,456.000000,2190.000000,2132.000000,2629.000000,3278.000000,1429.000000,489.000000,46.000000,441.000000,229.000000,181.000000,69.000000,69.000000,20.000000,119.000000,211.000000,63.000000,21.000000,1.000000,2.000000,2.000000,1.000000,2.000000,1.000000,1.000000
				]
			}
		]
	}
]
```
In text form:
```url
# numerical
http://localhost:51234/format('text');q(crimes.b('type',dive(1)),'name')
```
results in
```txt
                                              type                           count
                                         NARCOTICS                     5742.000000
                                   CRIMINAL DAMAGE                     4660.000000
                                             THEFT                    11367.000000
                                           BATTERY                     8990.000000
                                          BURGLARY                     2933.000000
                               MOTOR VEHICLE THEFT                     2226.000000
                        OFFENSE INVOLVING CHILDREN                      456.000000
                                DECEPTIVE PRACTICE                     2190.000000
                                           ROBBERY                     2132.000000
                                           ASSAULT                     2629.000000
                                     OTHER OFFENSE                     3278.000000
                                 CRIMINAL TRESPASS                     1429.000000
                                 WEAPONS VIOLATION                      489.000000
                                        KIDNAPPING                       46.000000
                            PUBLIC PEACE VIOLATION                      441.000000
                  INTERFERENCE WITH PUBLIC OFFICER                      229.000000
                               CRIM SEXUAL ASSAULT                      181.000000
                              LIQUOR LAW VIOLATION                       69.000000
                                          HOMICIDE                       69.000000
                                          STALKING                       20.000000
                                       SEX OFFENSE                      119.000000
                                      PROSTITUTION                      211.000000
                                             ARSON                       63.000000
                                      INTIMIDATION                       21.000000
                                      NON-CRIMINAL                        1.000000
                                  PUBLIC INDECENCY                        2.000000
                                          GAMBLING                        2.000000
                                         OBSCENITY                        1.000000
                          OTHER NARCOTIC VIOLATION                        2.000000
                 CONCEALED CARRY LICENSE VIOLATION                        1.000000
                  NON-CRIMINAL (SUBJECT SPECIFIED)                        1.000000
```

## time

Assuming a dimension of a nanocube is a binary tree, we can get aggregate values
for a sequence of fixed width (in the finest resolution of the tree) using the
`'intseq'` *target*.

```
#
# Usage intseq: intseq(OFFSET,WIDTH,COUNT[,STRIDE])
#
# In the example of crime50k the 'time' dimension was specified in
# the .map file as
#
# index_dimension('time',                         # dimension name
#                 input('Date'),                  # .csv column wher the input date of the record is specified
#                 time(16,                        # binary tree with 16 levels (slots 0 to 65535)
#                    '2013-12-01T00:00:00-06:00', # offset 0 corresponds to 0 hour of Dec 1, 2013 at CST (Central Standard Time)
#                    3600,                        # finest time bin corresponds to 1 hour (or 3600 secs)
#                    6*60                         # add 360 minutes or 6 hoursto to all input values since input comes as
#                                                 # if it is UTC while it is actually CST Central Standard Time
#                   ));
#
# So the time bin semantic is the following: there are 16 levels in the binary
# tree which yields a correspondence between its leaves and the numbers {0,1,...,65535}.
# By the `index_dimension` spec above, each of these numbers correspond to one hour
# aggregate of date (ie. 3600 secs). Tying back these (fine time resolution)
# numbers to the calendar, we have the following correspondence:
#
#     time interval                                             finest bin
#                                                               or leaf number
#     [2013-12-01T00:00:00-06:00,2013-12-01T00:00:00-06:00)     0
#     [2013-12-01T00:00:01-06:00,2013-12-01T01:00:00-06:00)     1
#     [2013-12-01T00:00:02-06:00,2013-12-01T02:00:00-06:00)     2
#     [2013-12-01T00:00:03-06:00,2013-12-01T03:00:00-06:00)     3
#     ...
#     [2013-12-21T00:00:03-06:00,2013-12-01T00:00:00-06:00)     480 (is 20 days later)
#     ...
#
# if we want to query 10 days of daily aggregates starting at the 20th day from
# the 0 time bin, we would like to aggregate
#
#     [480 + 00 * 24, 480 + 01 * 24)
#     [480 + 01 * 24, 480 + 02 * 24)
#     ...
#     [480 + 09 * 24, 480 + 10 * 24)
#
# we achieve that by setting
#
#     OFFSET = 480
#     WIDTH  = 24
#     COUNT  = 10
#     STRIDE = 24  (we can ommit STRIDE when it is the same as the WIDTH)
#
# so the query we wan here can be one of the two below
#
http://localhost:51234/q(crimes.b('time',intseq(480,24,10,24)))
http://localhost:51234/q(crimes.b('time',intseq(480,24,10)))
```
The result of the `'intseq'` query above is
```json
[
	{
		"type":"table",
		"numrows":10,
		"index_columns":[
			{
				"name":"time",
				"hint":"none",
				"values_per_row":1,
				"values":[
					0,1,2,3,4,5,6,7,8,9
				]
			}
		],
		"measure_columns":[
			{
				"name":"count",
				"values":[
					762.000000,724.000000,660.000000,515.000000,410.000000,584.000000,713.000000,712.000000,704.000000,617.000000
				]
			}
		]
	}
]
```











```
# Schema
http://localhost:29512/schema

{ "fields":[ { "name":"location", "type":"nc_dim_quadtree_25", "valnames":{  } }, { "name":"crime", "type":"nc_dim_cat_1", "valnames":{ "OTHER_OFFENSE":22, "NON-CRIMINAL_(SUBJECT_SPECIFIED)":18, "NARCOTICS":16, "GAMBLING":9, "MOTOR_VEHICLE_THEFT":15, "OTHER_NARCOTIC_VIOLATION":21, "OBSCENITY":19, "HOMICIDE":10, "THEFT":29, "DECEPTIVE_PRACTICE":8, "CRIMINAL_DAMAGE":5, "STALKING":28, "BATTERY":2, "PUBLIC_PEACE_VIOLATION":25, "PUBLIC_INDECENCY":24, "ASSAULT":1, "BURGLARY":3, "ROBBERY":26, "LIQUOR_LAW_VIOLATION":14, "INTERFERENCE_WITH_PUBLIC_OFFICER":11, "NON-CRIMINAL":17, "PROSTITUTION":23, "ARSON":0, "INTIMIDATION":12, "SEX_OFFENSE":27, "CONCEALED_CARRY_LICENSE_VIOLATION":4, "OFFENSE_INVOLVING_CHILDREN":20, "KIDNAPPING":13, "CRIM_SEXUAL_ASSAULT":7, "WEAPONS_VIOLATION":30, "CRIMINAL_TRESPASS":6 } }, { "name":"time", "type":"nc_dim_time_2", "valnames":{  } }, { "name":"count", "type":"nc_var_uint_4", "valnames":{  } } ], "metadata":[ { "key":"location__origin", "value":"degrees_mercator_quadtree25" }, { "key":"tbin", "value":"2013-12-01_00:00:00_3600s" }, { "key":"name", "value":"crime50k.csv" } ] }
```

## `.count`

Returns the sum of values in a certain product bin.

The following examples of API calls show the initial query and then the returned results.

```
## No constraints example
http://localhost:29512/count

{ "layers":[  ], "root":{ "val":50000 } }
```

```
## split on space (quadtree path 2,1,2) on a 256x256 image
http://localhost:29512/count.a("location",dive([2,1,2],8))

{ "layers":[ "anchor:location" ], "root":{ "children":[ { "path":[2,1,2,0,0,0,0,1,3,2,3], "val":7606 }, { "path":[2,1,2,0,0,0,0,1,3,1,2], "val":224 }, { "path":[2,1,2,0,0,0,0,1,2,3,3], "val":272 }, { "path":[2,1,2,0,0,0,0,1,3,2,2], "val":5461 }, { "path":[2,1,2,0,0,0,0,1,3,2,1], "val":16914 }, { "path":[2,1,2,0,0,0,0,1,3,2,0], "val":6839 }, { "path":[2,1,2,0,0,0,0,1,3,0,3], "val":12268 }, { "path":[2,1,2,0,0,0,0,1,3,0,2], "val":416 } ] } }
```

```
## restrict to a rectangular area (see figure below).  Specify lower-left and upper-right bounds.
http://localhost:29512/count.r("location",range2d(tile2d(1049,2571,12),tile2d(1050,2572,12)))

{ "layers":[  ], "root":{ "val":11167 } }
```

![image](./img/range-example.png)


```
## split on time: base time bin = 480, bucket has 24 bins, get 10 buckets (if they exist)
http://localhost:29512/count.r("time",mt_interval_sequence(480,24,10))

{ "layers":[ "multi-target:time" ], "root":{ "children":[ { "path":[0], "val":762 }, { "path":[1], "val":724 }, { "path":[2], "val":660 }, { "path":[3], "val":515 }, { "path":[4], "val":410 }, { "path":[5], "val":584 }, { "path":[6], "val":713 }, { "path":[7], "val":712 }, { "path":[8], "val":704 }, { "path":[9], "val":617 } ] } }
```

```
## combine location and time queries above (time series for each pixel)
http://localhost:29512/count.a("location",dive([2,1,2],8)).r("time",mt_interval_sequence(480,24,10))

{ "layers":[ "anchor:location", "multi-target:time" ], "root":{ "children":[ { "path":[2,1,2,0,0,0,0,1,3,2,3], "children":[ { "path":[0], "val":111 }, { "path":[1], "val":120 }, { "path":[2], "val":101 }, { "path":[3], "val":94 }, { "path":[4], "val":59 }, { "path":[5], "val":69 }, { "path":[6], "val":88 }, { "path":[7], "val":106 }, { "path":[8], "val":102 }, { "path":[9], "val":84 } ] }, { "path":[2,1,2,0,0,0,0,1,3,1,2], "children":[ { "path":[0], "val":4 }, { "path":[1], "val":1 }, { "path":[2], "val":2 }, { "path":[3], "val":1 }, { "path":[4], "val":5 }, { "path":[5], "val":2 }, { "path":[6], "val":3 }, { "path":[7], "val":5 }, { "path":[8], "val":3 }, { "path":[9], "val":1 } ] }, { "path":[2,1,2,0,0,0,0,1,2,3,3], "children":[ { "path":[0], "val":2 }, { "path":[1], "val":6 }, { "path":[2], "val":6 }, { "path":[3], "val":5 }, { "path":[4], "val":3 }, { "path":[5], "val":5 }, { "path":[6], "val":4 }, { "path":[7], "val":5 }, { "path":[8], "val":6 }, { "path":[9], "val":1 } ] }, { "path":[2,1,2,0,0,0,0,1,3,2,2], "children":[ { "path":[0], "val":103 }, { "path":[1], "val":77 }, { "path":[2], "val":65 }, { "path":[3], "val":52 }, { "path":[4], "val":62 }, { "path":[5], "val":72 }, { "path":[6], "val":88 }, { "path":[7], "val":80 }, { "path":[8], "val":76 }, { "path":[9], "val":58 } ] }, { "path":[2,1,2,0,0,0,0,1,3,2,1], "children":[ { "path":[0], "val":270 }, { "path":[1], "val":227 }, { "path":[2], "val":248 }, { "path":[3], "val":161 }, { "path":[4], "val":119 }, { "path":[5], "val":175 }, { "path":[6], "val":233 }, { "path":[7], "val":234 }, { "path":[8], "val":259 }, { "path":[9], "val":216 } ] }, { "path":[2,1,2,0,0,0,0,1,3,2,0], "children":[ { "path":[0], "val":101 }, { "path":[1], "val":93 }, { "path":[2], "val":86 }, { "path":[3], "val":61 }, { "path":[4], "val":71 }, { "path":[5], "val":104 }, { "path":[6], "val":106 }, { "path":[7], "val":117 }, { "path":[8], "val":86 }, { "path":[9], "val":93 } ] }, { "path":[2,1,2,0,0,0,0,1,3,0,3], "children":[ { "path":[0], "val":165 }, { "path":[1], "val":194 }, { "path":[2], "val":147 }, { "path":[3], "val":135 }, { "path":[4], "val":90 }, { "path":[5], "val":150 }, { "path":[6], "val":181 }, { "path":[7], "val":160 }, { "path":[8], "val":164 }, { "path":[9], "val":160 } ] }, { "path":[2,1,2,0,0,0,0,1,3,0,2], "children":[ { "path":[0], "val":6 }, { "path":[1], "val":6 }, { "path":[2], "val":5 }, { "path":[3], "val":6 }, { "path":[4], "val":1 }, { "path":[5], "val":7 }, { "path":[6], "val":10 }, { "path":[7], "val":5 }, { "path":[8], "val":8 }, { "path":[9], "val":4 } ] } ] } }
```

```
## tile2d example tile2d(x,y,level)... x,y in {0,...,2^level-1}x{0,...,2^level-1}
http://localhost:29512/count.a("location",dive(tile2d(1,2,2),8))

{ "layers":[ "anchor:location" ], "root":{ "children":[ { "path":[2,1,2,0,0,0,0,1,3,2], "val":36820 }, { "path":[2,1,2,0,0,0,0,1,3,1], "val":224 }, { "path":[2,1,2,0,0,0,0,1,3,0], "val":12684 }, { "path":[2,1,2,0,0,0,0,1,2,3], "val":272 } ] } }
```

```
## same as above, but passing the "img" formatting hint for bidimensional image addresses (relative address to tile(1,2,2))
http://localhost:29512/count.a("location",dive(tile2d(1,2,2),8),"img")

{ "layers":[ "anchor:location" ], "root":{ "children":[ { "x":6, "y":131, "val":36820 }, { "x":7, "y":130, "val":224 }, { "x":6, "y":130, "val":12684 }, { "x":5, "y":131, "val":272 } ] } }
```

```
## branch on the "crime" type
http://localhost:29512/count.a("crime",dive([],1))

{ "layers":[ "anchor:crime" ], "root":{ "children":[ { "path":[21], "val":2 }, { "path":[20], "val":456 }, { "path":[19], "val":1 }, { "path":[18], "val":1 }, { "path":[17], "val":1 }, { "path":[16], "val":5742 }, { "path":[15], "val":2226 }, { "path":[14], "val":69 }, { "path":[13], "val":46 }, { "path":[12], "val":21 }, { "path":[11], "val":229 }, { "path":[10], "val":69 }, { "path":[0], "val":63 }, { "path":[1], "val":2629 }, { "path":[2], "val":8990 }, { "path":[3], "val":2933 }, { "path":[4], "val":1 }, { "path":[5], "val":4660 }, { "path":[6], "val":1429 }, { "path":[7], "val":181 }, { "path":[8], "val":2190 }, { "path":[9], "val":2 }, { "path":[22], "val":3278 }, { "path":[23], "val":211 }, { "path":[24], "val":2 }, { "path":[25], "val":441 }, { "path":[26], "val":2132 }, { "path":[27], "val":119 }, { "path":[28], "val":20 }, { "path":[29], "val":11367 }, { "path":[30], "val":489 } ] } }
```

```
## set target
http://localhost:29512/count.r("crime",set([1],[3]))

{ "layers":[  ], "root":{ "val":5562 } }
```

```
## ... or a shortcut for addresses in the first level of a binning structure
http://localhost:29512/count.r("crime",set(1,3))

{ "layers":[  ], "root":{ "val":5562 } }
```

```
## image restricted to a time interval [a,b]
http://localhost:29512/count.r("time",interval(484,500)).a("location",dive(tile2d(262,643,10),1),"img")

{ "layers":[ "anchor:location" ], "root":{ "children":[ { "x":1, "y":1, "val":67 }, { "x":1, "y":0, "val":188 }, { "x":0, "y":1, "val":65 }, { "x":0, "y":0, "val":74 } ] } }
```

```
## a time series of images (time series for each pixel)
http://localhost:29512/count.r("time",mt_interval_sequence(484,1,5)).a("location",dive(tile2d(262,643,10),1),"img")

{ "layers":[ "anchor:location", "multi-target:time" ], "root":{ "children":[ { "x":1, "y":1, "children":[ { "path":[0], "val":5 }, { "path":[1], "val":2 }, { "path":[2], "val":1 }, { "path":[4], "val":1 } ] }, { "x":1, "y":0, "children":[ { "path":[0], "val":1 }, { "path":[1], "val":3 }, { "path":[2], "val":6 }, { "path":[3], "val":4 }, { "path":[4], "val":11 } ] }, { "x":0, "y":1, "children":[ { "path":[0], "val":1 }, { "path":[1], "val":3 }, { "path":[2], "val":1 }, { "path":[3], "val":3 }, { "path":[4], "val":1 } ] }, { "x":0, "y":0, "children":[ { "path":[0], "val":3 }, { "path":[1], "val":1 }, { "path":[3], "val":3 }, { "path":[4], "val":2 } ] } ] } }
```

```
## degrees_mask (longitude,latitude) single contour
http://localhost:29512/count.r("location",degrees_mask("-87.6512,41.8637,-87.6512,41.9009,-87.6026,41.9009,-87.6026,41.8637",25))

{ "layers":[  ], "root":{ "val":3344 } }
```

## unsorted
http://localhost:29512/count.a("location",mask("012<<12<<<",10))
http://localhost:29512/count.a("location",region("us_states/newyork",10))   // search directory
http://localhost:29512/count.a("location",mercator_mask("x0,y0,x1,y1,...,xn,yn;x0,y0,x1,y1,x2,y2",10))
http://localhost:29512/count.a("location",degrees_mask("x0,y0,x1,y1,...,xn,yn;x0,y0,x1,y1,x2,y2",10))


## `.timing`

Timing

## `.shutdown`

To remotely shutdown a running nanocube server, use the shutdown service.  To provide some level of security,
we have provided a passcode (standard out) when the nanocube server was started.  The passcode must be provided
when requesting a shutdown:

http://localhost:29512/shutdown.passcode("abcdefgh")

If you have included the correct passcode, you will see the following message:

`Nanocube server authorized shutdown in progress`

Otherwise, you will see the following:

`Ignoring unauthorized Nanocube server shutdown`


### Output Encoding

There are three kinds of encodings: json (default), text, and
binary. To activate these methods (the last activated will be used)
three functions are avaiable on the queries: `.json()`, `.text()`,
`.bin()`.

# ToDo

- Access of Multiple Variables






<!-- sophia@oreilly.com -->

<!--     server.port = options.query_port.getValue(); -->
   
<!--     bool json        = true; -->
<!--     bool binary      = false; -->
<!--     bool compression = true; -->
<!--     bool plain       = false; -->
   
<!--     auto json_query_handler    = std::bind(&NanocubeServer::serveQuery, this, std::placeholders::_1, json,       plain); -->
<!--     auto binary_query_handler  = std::bind(&NanocubeServer::serveQuery, this, std::placeholders::_1, binary,     plain); -->
<!--     auto json_tquery_handler   = std::bind(&NanocubeServer::serveTimeQuery, this, std::placeholders::_1, json,   plain); -->
<!--     auto binary_tquery_handler = std::bind(&NanocubeServer::serveTimeQuery, this, std::placeholders::_1, binary, plain); -->
<!--     // auto json_query_comp_handler    = std::bind(&NanocubeServer::serveQuery, this, std::placeholders::_1, json,       compression); -->
<!--     // auto json_tquery_comp_handler   = std::bind(&NanocubeServer::serveTimeQuery, this, std::placeholders::_1, json,   compression); -->
<!--     auto binary_query_comp_handler  = std::bind(&NanocubeServer::serveQuery, this, std::placeholders::_1, binary,     compression); -->
<!--     auto binary_tquery_comp_handler = std::bind(&NanocubeServer::serveTimeQuery, this, std::placeholders::_1, binary, compression); -->
<!--     auto stats_handler         = std::bind(&NanocubeServer::serveStats, this, std::placeholders::_1); -->
<!--     auto binary_schema_handler = std::bind(&NanocubeServer::serveSchema,     this, std::placeholders::_1, binary); -->
<!--     auto schema_handler        = std::bind(&NanocubeServer::serveSchema,     this, std::placeholders::_1, json); -->
<!--     auto valname_handler       = std::bind(&NanocubeServer::serveSetValname, this, std::placeholders::_1); -->
<!--     auto version_handler       = std::bind(&NanocubeServer::serveVersion,    this, std::placeholders::_1); -->
<!--     auto tbin_handler          = std::bind(&NanocubeServer::serveTBin, this, std::placeholders::_1); -->
<!--     auto summary_handler       = std::bind(&NanocubeServer::serveSummary, this, std::placeholders::_1); -->
<!--     auto graphviz_handler      = std::bind(&NanocubeServer::serveGraphViz, this, std::placeholders::_1); -->
<!--     auto timing_handler        = std::bind(&NanocubeServer::serveTiming, this, std::placeholders::_1); -->
<!--     auto tile_handler         = std::bind(&NanocubeServer::serveTile, this, std::placeholders::_1); -->
   









<!-- # API 1 -->

<!-- http://nanocubes.net/nanocube/14/tile/4/8/7/10/0/10000000000/ -->
<!-- http://nanocubes.net/nanocube/14/query/region/0/0/0/1/1/where/hour_of_day=05 -->

<!-- # API 2 -->

<!-- http://lion5.research.att.com:29527/query/time=16224:8392:1/src=<qaddr(999,829,10),qaddr(0,829,10),qaddr(0,460,10),qaddr(999,460,10)>/@device=255+1 -->

<!-- # API 3 -->



To illustrate the internals of a nanocube, we use the example in Figure 2 of the
original [Nanocubes technical paper](http://nanocubes.net/assets/pdf/nanocubes_paper.pdf).

```shell
# using permutation (location, device)
nanocube paper.csv paper_location_device.map paper_location_device.nanocube
# using permutation (device, location)
nanocube paper.csv paper_device_location.map paper_device_location.nanocube
```



















# Dimensions and Bins

Nanocubes is all about binning multi-dimenstional data. As a running
example, let's consider a table like the Chicago crime one:

    latitude | longitude | timestamp | type

How many records in the geo-location table fall into a certain spatial
bin? Or is of type `theft`? By efficiently pre-computing and storing
counts or measures, nanocube can handle a good amount of records in a
few dimensions (e.g. one or two spatial dimensions, one temporal dimension,
a few categorical dimensions) to solve queries at interactive rates.

# Paths

A *path* is the identifier of a *bin* in a *dimension* of a
nanocube. This *bin* can be either an aggregate one or a finest
resolution one. For example, if we have a dimension for a categorical
variable "device" represented as a two level tree, the root represents
any device and a leaf might represent an `iPhone` or `Android`.

# Target

A *target* for a dimension restricts the set of records of interest to
the ones that are in a particular set of *bins* in that dimension. For
example, if we have a categorical dimension with US States as *bins*,
we can think of `{NY, LA, UT}` as a *target* for that dimension. More
formally, a *target* for dimension *d* specifies a set of *paths* that
should be visited every time the execution engine that is solving a
query needs to traverse a *binning hierarchy* on dimensions *d*. If a
certain *path* in the *target* is not present for a particular *bin
hierarchy* instance, then, obviously, this path will not be visited.

## Multi-Target

Sometimes we want to collect separetely values of multiple targets on
a single dimension. For example, we might want to query multiple
consecutive intervals from a binary tree representation for time. Each
interval data can be "solved" by visiting a (minimal) set of time bins
that is a cover for it (the interval).

# Services

## `.schema`

The schema reports all of the dimensions (i.e. fields) of the nanocube
and their types.

In this example, there are four fields with names:
`location`, `crime`, `time`, and `count`.  The types of these fields
are (roughly) described as: quadree with 25 levels, categorical with 1
byte, time with 2 bytes, and unsigned integer with 4 bytes. Also
specified for each field are the valid values of these fields.  In
this example, we only really need to specify them for the categorical
dimension `crime` which lists the names and values of the criminal
offenses. There is also some additional metadata reported to indicate
when the time dimension should begin, how large the time bins are, how
to project the quadtree onto a map, and the original data file name.

```
# Schema
http://localhost:29512/schema

{ "fields":[ { "name":"location", "type":"nc_dim_quadtree_25", "valnames":{  } }, { "name":"crime", "type":"nc_dim_cat_1", "valnames":{ "OTHER_OFFENSE":22, "NON-CRIMINAL_(SUBJECT_SPECIFIED)":18, "NARCOTICS":16, "GAMBLING":9, "MOTOR_VEHICLE_THEFT":15, "OTHER_NARCOTIC_VIOLATION":21, "OBSCENITY":19, "HOMICIDE":10, "THEFT":29, "DECEPTIVE_PRACTICE":8, "CRIMINAL_DAMAGE":5, "STALKING":28, "BATTERY":2, "PUBLIC_PEACE_VIOLATION":25, "PUBLIC_INDECENCY":24, "ASSAULT":1, "BURGLARY":3, "ROBBERY":26, "LIQUOR_LAW_VIOLATION":14, "INTERFERENCE_WITH_PUBLIC_OFFICER":11, "NON-CRIMINAL":17, "PROSTITUTION":23, "ARSON":0, "INTIMIDATION":12, "SEX_OFFENSE":27, "CONCEALED_CARRY_LICENSE_VIOLATION":4, "OFFENSE_INVOLVING_CHILDREN":20, "KIDNAPPING":13, "CRIM_SEXUAL_ASSAULT":7, "WEAPONS_VIOLATION":30, "CRIMINAL_TRESPASS":6 } }, { "name":"time", "type":"nc_dim_time_2", "valnames":{  } }, { "name":"count", "type":"nc_var_uint_4", "valnames":{  } } ], "metadata":[ { "key":"location__origin", "value":"degrees_mercator_quadtree25" }, { "key":"tbin", "value":"2013-12-01_00:00:00_3600s" }, { "key":"name", "value":"crime50k.csv" } ] }
```

## `.count`

Returns the sum of values in a certain product bin.

The following examples of API calls show the initial query and then the returned results.

```
## No constraints example
http://localhost:29512/count

{ "layers":[  ], "root":{ "val":50000 } }
```

```
## split on space (quadtree path 2,1,2) on a 256x256 image
http://localhost:29512/count.a("location",dive([2,1,2],8))

{ "layers":[ "anchor:location" ], "root":{ "children":[ { "path":[2,1,2,0,0,0,0,1,3,2,3], "val":7606 }, { "path":[2,1,2,0,0,0,0,1,3,1,2], "val":224 }, { "path":[2,1,2,0,0,0,0,1,2,3,3], "val":272 }, { "path":[2,1,2,0,0,0,0,1,3,2,2], "val":5461 }, { "path":[2,1,2,0,0,0,0,1,3,2,1], "val":16914 }, { "path":[2,1,2,0,0,0,0,1,3,2,0], "val":6839 }, { "path":[2,1,2,0,0,0,0,1,3,0,3], "val":12268 }, { "path":[2,1,2,0,0,0,0,1,3,0,2], "val":416 } ] } }
```

```
## restrict to a rectangular area (see figure below).  Specify lower-left and upper-right bounds.
http://localhost:29512/count.r("location",range2d(tile2d(1049,2571,12),tile2d(1050,2572,12)))

{ "layers":[  ], "root":{ "val":11167 } }
```

![image](./img/range-example.png)


```
## split on time: base time bin = 480, bucket has 24 bins, get 10 buckets (if they exist)
http://localhost:29512/count.r("time",mt_interval_sequence(480,24,10))

{ "layers":[ "multi-target:time" ], "root":{ "children":[ { "path":[0], "val":762 }, { "path":[1], "val":724 }, { "path":[2], "val":660 }, { "path":[3], "val":515 }, { "path":[4], "val":410 }, { "path":[5], "val":584 }, { "path":[6], "val":713 }, { "path":[7], "val":712 }, { "path":[8], "val":704 }, { "path":[9], "val":617 } ] } }
```

```
## combine location and time queries above (time series for each pixel)
http://localhost:29512/count.a("location",dive([2,1,2],8)).r("time",mt_interval_sequence(480,24,10))

{ "layers":[ "anchor:location", "multi-target:time" ], "root":{ "children":[ { "path":[2,1,2,0,0,0,0,1,3,2,3], "children":[ { "path":[0], "val":111 }, { "path":[1], "val":120 }, { "path":[2], "val":101 }, { "path":[3], "val":94 }, { "path":[4], "val":59 }, { "path":[5], "val":69 }, { "path":[6], "val":88 }, { "path":[7], "val":106 }, { "path":[8], "val":102 }, { "path":[9], "val":84 } ] }, { "path":[2,1,2,0,0,0,0,1,3,1,2], "children":[ { "path":[0], "val":4 }, { "path":[1], "val":1 }, { "path":[2], "val":2 }, { "path":[3], "val":1 }, { "path":[4], "val":5 }, { "path":[5], "val":2 }, { "path":[6], "val":3 }, { "path":[7], "val":5 }, { "path":[8], "val":3 }, { "path":[9], "val":1 } ] }, { "path":[2,1,2,0,0,0,0,1,2,3,3], "children":[ { "path":[0], "val":2 }, { "path":[1], "val":6 }, { "path":[2], "val":6 }, { "path":[3], "val":5 }, { "path":[4], "val":3 }, { "path":[5], "val":5 }, { "path":[6], "val":4 }, { "path":[7], "val":5 }, { "path":[8], "val":6 }, { "path":[9], "val":1 } ] }, { "path":[2,1,2,0,0,0,0,1,3,2,2], "children":[ { "path":[0], "val":103 }, { "path":[1], "val":77 }, { "path":[2], "val":65 }, { "path":[3], "val":52 }, { "path":[4], "val":62 }, { "path":[5], "val":72 }, { "path":[6], "val":88 }, { "path":[7], "val":80 }, { "path":[8], "val":76 }, { "path":[9], "val":58 } ] }, { "path":[2,1,2,0,0,0,0,1,3,2,1], "children":[ { "path":[0], "val":270 }, { "path":[1], "val":227 }, { "path":[2], "val":248 }, { "path":[3], "val":161 }, { "path":[4], "val":119 }, { "path":[5], "val":175 }, { "path":[6], "val":233 }, { "path":[7], "val":234 }, { "path":[8], "val":259 }, { "path":[9], "val":216 } ] }, { "path":[2,1,2,0,0,0,0,1,3,2,0], "children":[ { "path":[0], "val":101 }, { "path":[1], "val":93 }, { "path":[2], "val":86 }, { "path":[3], "val":61 }, { "path":[4], "val":71 }, { "path":[5], "val":104 }, { "path":[6], "val":106 }, { "path":[7], "val":117 }, { "path":[8], "val":86 }, { "path":[9], "val":93 } ] }, { "path":[2,1,2,0,0,0,0,1,3,0,3], "children":[ { "path":[0], "val":165 }, { "path":[1], "val":194 }, { "path":[2], "val":147 }, { "path":[3], "val":135 }, { "path":[4], "val":90 }, { "path":[5], "val":150 }, { "path":[6], "val":181 }, { "path":[7], "val":160 }, { "path":[8], "val":164 }, { "path":[9], "val":160 } ] }, { "path":[2,1,2,0,0,0,0,1,3,0,2], "children":[ { "path":[0], "val":6 }, { "path":[1], "val":6 }, { "path":[2], "val":5 }, { "path":[3], "val":6 }, { "path":[4], "val":1 }, { "path":[5], "val":7 }, { "path":[6], "val":10 }, { "path":[7], "val":5 }, { "path":[8], "val":8 }, { "path":[9], "val":4 } ] } ] } }
```

```
## tile2d example tile2d(x,y,level)... x,y in {0,...,2^level-1}x{0,...,2^level-1}
http://localhost:29512/count.a("location",dive(tile2d(1,2,2),8))

{ "layers":[ "anchor:location" ], "root":{ "children":[ { "path":[2,1,2,0,0,0,0,1,3,2], "val":36820 }, { "path":[2,1,2,0,0,0,0,1,3,1], "val":224 }, { "path":[2,1,2,0,0,0,0,1,3,0], "val":12684 }, { "path":[2,1,2,0,0,0,0,1,2,3], "val":272 } ] } }
```

```
## same as above, but passing the "img" formatting hint for bidimensional image addresses (relative address to tile(1,2,2))
http://localhost:29512/count.a("location",dive(tile2d(1,2,2),8),"img")

{ "layers":[ "anchor:location" ], "root":{ "children":[ { "x":6, "y":131, "val":36820 }, { "x":7, "y":130, "val":224 }, { "x":6, "y":130, "val":12684 }, { "x":5, "y":131, "val":272 } ] } }
```

```
## branch on the "crime" type
http://localhost:29512/count.a("crime",dive([],1))

{ "layers":[ "anchor:crime" ], "root":{ "children":[ { "path":[21], "val":2 }, { "path":[20], "val":456 }, { "path":[19], "val":1 }, { "path":[18], "val":1 }, { "path":[17], "val":1 }, { "path":[16], "val":5742 }, { "path":[15], "val":2226 }, { "path":[14], "val":69 }, { "path":[13], "val":46 }, { "path":[12], "val":21 }, { "path":[11], "val":229 }, { "path":[10], "val":69 }, { "path":[0], "val":63 }, { "path":[1], "val":2629 }, { "path":[2], "val":8990 }, { "path":[3], "val":2933 }, { "path":[4], "val":1 }, { "path":[5], "val":4660 }, { "path":[6], "val":1429 }, { "path":[7], "val":181 }, { "path":[8], "val":2190 }, { "path":[9], "val":2 }, { "path":[22], "val":3278 }, { "path":[23], "val":211 }, { "path":[24], "val":2 }, { "path":[25], "val":441 }, { "path":[26], "val":2132 }, { "path":[27], "val":119 }, { "path":[28], "val":20 }, { "path":[29], "val":11367 }, { "path":[30], "val":489 } ] } }
```

```
## set target
http://localhost:29512/count.r("crime",set([1],[3]))

{ "layers":[  ], "root":{ "val":5562 } }
```

```
## ... or a shortcut for addresses in the first level of a binning structure
http://localhost:29512/count.r("crime",set(1,3))

{ "layers":[  ], "root":{ "val":5562 } }
```

```
## image restricted to a time interval [a,b]
http://localhost:29512/count.r("time",interval(484,500)).a("location",dive(tile2d(262,643,10),1),"img")

{ "layers":[ "anchor:location" ], "root":{ "children":[ { "x":1, "y":1, "val":67 }, { "x":1, "y":0, "val":188 }, { "x":0, "y":1, "val":65 }, { "x":0, "y":0, "val":74 } ] } }
```

```
## a time series of images (time series for each pixel)
http://localhost:29512/count.r("time",mt_interval_sequence(484,1,5)).a("location",dive(tile2d(262,643,10),1),"img")

{ "layers":[ "anchor:location", "multi-target:time" ], "root":{ "children":[ { "x":1, "y":1, "children":[ { "path":[0], "val":5 }, { "path":[1], "val":2 }, { "path":[2], "val":1 }, { "path":[4], "val":1 } ] }, { "x":1, "y":0, "children":[ { "path":[0], "val":1 }, { "path":[1], "val":3 }, { "path":[2], "val":6 }, { "path":[3], "val":4 }, { "path":[4], "val":11 } ] }, { "x":0, "y":1, "children":[ { "path":[0], "val":1 }, { "path":[1], "val":3 }, { "path":[2], "val":1 }, { "path":[3], "val":3 }, { "path":[4], "val":1 } ] }, { "x":0, "y":0, "children":[ { "path":[0], "val":3 }, { "path":[1], "val":1 }, { "path":[3], "val":3 }, { "path":[4], "val":2 } ] } ] } }
```

```
## degrees_mask (longitude,latitude) single contour
http://localhost:29512/count.r("location",degrees_mask("-87.6512,41.8637,-87.6512,41.9009,-87.6026,41.9009,-87.6026,41.8637",25))

{ "layers":[  ], "root":{ "val":3344 } }
```

## unsorted
http://localhost:29512/count.a("location",mask("012<<12<<<",10))
http://localhost:29512/count.a("location",region("us_states/newyork",10))   // search directory
http://localhost:29512/count.a("location",mercator_mask("x0,y0,x1,y1,...,xn,yn;x0,y0,x1,y1,x2,y2",10))
http://localhost:29512/count.a("location",degrees_mask("x0,y0,x1,y1,...,xn,yn;x0,y0,x1,y1,x2,y2",10))


## `.timing`

Timing

## `.shutdown`

To remotely shutdown a running nanocube server, use the shutdown service.  To provide some level of security,
we have provided a passcode (standard out) when the nanocube server was started.  The passcode must be provided
when requesting a shutdown:

http://localhost:29512/shutdown.passcode("abcdefgh")

If you have included the correct passcode, you will see the following message:

`Nanocube server authorized shutdown in progress`

Otherwise, you will see the following:

`Ignoring unauthorized Nanocube server shutdown`


### Output Encoding

There are three kinds of encodings: json (default), text, and
binary. To activate these methods (the last activated will be used)
three functions are avaiable on the queries: `.json()`, `.text()`,
`.bin()`.

# ToDo

- Access of Multiple Variables






<!-- sophia@oreilly.com -->

<!--     server.port = options.query_port.getValue(); -->
   
<!--     bool json        = true; -->
<!--     bool binary      = false; -->
<!--     bool compression = true; -->
<!--     bool plain       = false; -->
   
<!--     auto json_query_handler    = std::bind(&NanocubeServer::serveQuery, this, std::placeholders::_1, json,       plain); -->
<!--     auto binary_query_handler  = std::bind(&NanocubeServer::serveQuery, this, std::placeholders::_1, binary,     plain); -->
<!--     auto json_tquery_handler   = std::bind(&NanocubeServer::serveTimeQuery, this, std::placeholders::_1, json,   plain); -->
<!--     auto binary_tquery_handler = std::bind(&NanocubeServer::serveTimeQuery, this, std::placeholders::_1, binary, plain); -->
<!--     // auto json_query_comp_handler    = std::bind(&NanocubeServer::serveQuery, this, std::placeholders::_1, json,       compression); -->
<!--     // auto json_tquery_comp_handler   = std::bind(&NanocubeServer::serveTimeQuery, this, std::placeholders::_1, json,   compression); -->
<!--     auto binary_query_comp_handler  = std::bind(&NanocubeServer::serveQuery, this, std::placeholders::_1, binary,     compression); -->
<!--     auto binary_tquery_comp_handler = std::bind(&NanocubeServer::serveTimeQuery, this, std::placeholders::_1, binary, compression); -->
<!--     auto stats_handler         = std::bind(&NanocubeServer::serveStats, this, std::placeholders::_1); -->
<!--     auto binary_schema_handler = std::bind(&NanocubeServer::serveSchema,     this, std::placeholders::_1, binary); -->
<!--     auto schema_handler        = std::bind(&NanocubeServer::serveSchema,     this, std::placeholders::_1, json); -->
<!--     auto valname_handler       = std::bind(&NanocubeServer::serveSetValname, this, std::placeholders::_1); -->
<!--     auto version_handler       = std::bind(&NanocubeServer::serveVersion,    this, std::placeholders::_1); -->
<!--     auto tbin_handler          = std::bind(&NanocubeServer::serveTBin, this, std::placeholders::_1); -->
<!--     auto summary_handler       = std::bind(&NanocubeServer::serveSummary, this, std::placeholders::_1); -->
<!--     auto graphviz_handler      = std::bind(&NanocubeServer::serveGraphViz, this, std::placeholders::_1); -->
<!--     auto timing_handler        = std::bind(&NanocubeServer::serveTiming, this, std::placeholders::_1); -->
<!--     auto tile_handler         = std::bind(&NanocubeServer::serveTile, this, std::placeholders::_1); -->
   









<!-- # API 1 -->

<!-- http://nanocubes.net/nanocube/14/tile/4/8/7/10/0/10000000000/ -->
<!-- http://nanocubes.net/nanocube/14/query/region/0/0/0/1/1/where/hour_of_day=05 -->

<!-- # API 2 -->

<!-- http://lion5.research.att.com:29527/query/time=16224:8392:1/src=<qaddr(999,829,10),qaddr(0,829,10),qaddr(0,460,10),qaddr(999,460,10)>/@device=255+1 -->

<!-- # API 3 -->
