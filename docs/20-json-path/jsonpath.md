## jsonpath disctionaries

```
{
  "vehicle":{
    "car": {
      "color": "blue",
      "price": "$20,000"
    },
  "bus": {
      "color": "white",
      "price": "$120,000"
   }
 }
}
```

# query [result will always be in square bracket]
$.vehicles.car
Output:
[
  {
      "color": "blue",
      "price": "$20,000"
   } 
]

$.vehicles.bus
Output:
[
  {
      "color": "white",
      "price": "$120,000"
   }
]

$.vehicles.car.color
[
 "blue"
]

$.vehicles.bus.price
[
 "$120,000"
]


## jsonpath lists
[
  "car",
  "bus",
  "truck",
  "bike"
]

# query
$[0]
output:
[ "car" ]

$[3]
output:
[ "bike" ]

$[0,3]
output:
[ "car", "bike" ]

## jsonpath dictionaries & lists
{
 "car": {
    "color": "blue",
    "price": "$20,000",
    "wheels": [
      {
         "model": "X345ERT"
         "location": "front-right"
      },
      {
         "model": "X346GRX"
         "location": "front-left"
      },
      {
         "model": "X236DEM"
         "location": "rear-right"
      },
      {
         "model": "X987XMV"
         "location": "rear-left"
      }
    ]
  }
}

# query
$.car.wheels[1].model
output:
[
 "X346GRX"
]

## jsonpath criteria
example1:
[
 12,
 43,
 23,
 12,
 56,
 43,
 93,
 32,
 45,
 63,
 27,
 8,
 78
]

#query
1: list numbers greater than 40
$[?( @>40 )]

output:
[
 43,
 56,
 43,
 93,
 45,
 63,
 78
]

Note: we can use other operators as well such as @==40 , @!=40, @ in [40,43,45], @ nin [40,43,45]

example2:
{
 "car": {
    "color": "blue",
    "price": "$20,000",
    "wheels": [
      {
         "model": "X345ERT"
         "location": "front-right"
      },
      {
         "model": "X346GRX"
         "location": "front-left"
      },
      {
         "model": "X236DEM"
         "location": "rear-right"
      },
      {
         "model": "X987XMV"
         "location": "rear-left"
      }
    ]
  }
}

#query:
2. suppose i want output "X236DEM" without specifying the exact index number
$.car.wheels[ ?( @.location == "rear-right" ) ].model

output:
[
 "X236DEM"
]


## jsonpath wildcard
Example 1:
{
  "car": {
      "color": "blue",
      "price": "$20,000"
    },
  "bus": {
      "color": "white",
      "price": "$120,000"
   }
}

1. Get all the colors of all the vehicles
# query
$.*.color

output:
[ "blue", "white"]

2. Get all the prices of all the vehicles
# query
$.*.price
[ "$20,000", "$120,000" ]

Example2:
[
      {
         "model": "X345ERT"
         "location": "front-right"
      },
      {
         "model": "X346GRX"
         "location": "front-left"
      },
      {
         "model": "X236DEM"
         "location": "rear-right"
      },
      {
         "model": "X987XMV"
         "location": "rear-left"
      } 
]

1. Get 1st wheel model
# quert
$[1].model

output:
["X345ERT"]

2. Get all wheels models
# query
$[*].model

output:
["X345ERT", "X346GRX", "X236DEM", "X987XMV"]

Example3:
{
 "car": {
    "color": "blue",
    "price": "$20,000",
    "wheels": [
      {
         "model": "X345ERT"
      },
      {
         "model": "X346GRX"
      },
    ]
  },
  "bus": {
    "color": "white",
    "price": "$120,000",
    "wheels": [
      {
         "model": "Z227KLJ"
      },
      {
         "model": "Z226KLJ"
      },
    ]
  }
}

1. Get car's 1st wheel model
$.car.wheels[0].model
output: [ "Z227KLJ" ]

2.Get car's all wheel models
$.car.wheels[*].model
output: 
[ "X345ERT" , "X346GRX" ]

3. Get bus all wheels models
$.bus.wheels[*].model
output:
["Z227KLJ", "Z226KLJ"]

4. Get models for all the car and bus
$.*.wheels[*].model
output: ["X345ERT", "X346GRX", "Z227KLJ", "Z226KLJ"]


## jsonpath lists
[
 "Apple",
 "Google",
 "Microsoft",
 "Amazon",
 "Facebook",
 "Coca-cola",
 "Samsung",
 "Disney",
 "Toyota",
 "McDonald's"
]

1. To get the first element
$[0]
output: ["Apple"]

2. To get the fourth element
$[3]
output: ["Amazon"]

3. To get the 1st and 4th element
$[0,3]
output: ["Apple", "Amazon"]

4. To get 1st to 4th element ( It will not include 4th element )
$[0:3]
output: ["Apple", "Google", "Microsoft"]

5. To get 1st to 4th element ( If want to include 4th element )
$[0:4]
output: ["Apple", "Google", "Microsoft", "Amazon"]

6. If want to skip 1 element and get the details
$[0:8:2]
output: ["Apple", "Microsoft", "Facebook", "Samsung"]

7. Get the last element
$[9]
output: ["McDonald's"]
$[-1]
output: ["McDonald's"]
$[-1:0]
output: ["McDonald's"]
$[-1:]
output: ["McDonald's"]

8. Get the last 3 elements
$[-3:]
output: ["McDonald's", "Toyota", "Disney"]


## why jsonpath
Large data sets
 * 100s of nodes
 * 1000's of pods, Deployments and replicaSets

kubectl utility when we run any commands communicate with kubeapi server and it returns the output in json format then kubectl utility convert it into
human readable format.

1. To get names of the nodes
k get nodes -o=jsonpath='{ .items[*].metadata.name }'

2. To get architecture used by nodes
k get nodes -o=jsonpath='{.items[*].status.nodeInfo.architecture}'

3. To get count of cpu's on the nodes
k get nodes -o=jsonpath='{.items[*].status.capacity.cpu}'

4. If want to get name of the node and cpu allocated to it want in the same command
k get nodes -o=jsonpath='{.items[*].metadata.name} {.items[*].status.capacity.cpu}'

# Above output is not readable, to make it readable we can use in bwtween 2 queried {"\n"} for newline and {"\t"} for tab
exa: k get nodes -o=jsonpath='{.items[*].metadata.name} {"\n"} {.items[*].status.capacity.cpu}'

# Loops range [ in this range means for each exa: for each node, for each pod etc]
'{range .items[*]}
   { .metadata.name } {"\t"} { .status.capacity.cpu } {"\n"}
{end}'

concert this into 1 line and pass to the command
k get nodes -o=jsonpath='{range .items[*]} { .metadata.name } {"\t"} { .status.capacity.cpu } {"\n"} {end}'

output will look like this
  gke-n7-playground-cl-nvizstore-solr-p-86743cd7-fqae    1
  gke-n7-playground-cl-nvizstore-solr-p-86743cd7-kcxg    1
  gke-n7-playground-cl-nvizstore-solr-p-86743cd7-lnk5    1
  gke-n7-playground-clu-custom-nodepool-f5cdef3d-qjcb    2

# we can use custom column option with kubectl command
k get nodes -o=custom-columns=<column name>:<json path>
k get nodes -o=custom-columns=NODE:.metadata.name
Note: In this no need to mention .items as it will assume it is for each item in the list

Note: we can add multiple columns accordingly
k get nodes -o=custom-columns=NODE:.metadata.name,CPU:.status.capacity.cpu
output:
NODE                                                  CPU
gke-n7-playground-cl-nvizstore-solr-p-86743cd7-fqae   1
gke-n7-playground-cl-nvizstore-solr-p-86743cd7-kcxg   1
gke-n7-playground-cl-nvizstore-solr-p-86743cd7-lnk5   1

# jsonpath for sort
k get nodes --sort-by=.metadata.name
k get nodes --sort-by=.status.capacity.cpu


