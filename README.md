
# GOOGLE SUMMER OF CODE: MAY 2020 - AUG 2020

## About Me

+ Name: Jay Batra
+ Github: [Jay](https://github.com/jaybatra26)
+ Email: jaybatra73@gmail.com
+ Slack nick: Jay Batra
+ Site: http://jaybatra26.github.io/
+ Mentor: [Bryan Boreham](https://twitter.com/bboreham)

## Selective Indexing of Labels

Selective indexing of labels:
The idea is to reduce the IO cost and storage cost by excluding some labels which are not very selective in nature. This would require adding a provision in the config file example a section named exclude_labels. While ingesting, this config is used to make sure that the ignored keywords are not ingested.
Design Doc: https://github.com/cortexproject/cortex/issues/2910

### Analysis tool to measure frequency of labels 

1. Implement API endpoint to measure selectivity of labels in existing data.
2. Implement measurement of frequency of labels in queries.
3. Design a way to store and access metadata on indexing.
4. Design a way to measure the effectiveness of the finished project.

The above points required deep understanding of Cortex Service.


#### Implementation:
The first task was to print the labels to identify how exactly how they look like? Initial implementation involved saving this into a CSV. A small script was written which fired all kinds of Prometheus queries called as [fire_queries.sh](https://github.com/jaybatra26/cortex/blob/leastlabel/fire_queries.sh)
The next step was to count the least occurring labels. The code for the same goes into `pkg/chunk/composite_store.go`. 
To get the information. First, Cortex is to be run, and `/flush` endpoint should be called. This has to be done because the default time of flush is 24hrs. 
Once flushed, use Oneshot program(helper program to call single query). Oneshot program is not present in the master branch, but is a part of the branch `oneshot` in the Cortex Project. One has to pull the code from oneshot branch

    `
    git checkout -b oneshot

    git pull origin oneshot

    go build ./cmd/oneshot
    
    ./oneshot -as-at 2020-08-31T04:22:56Z  -config.file=./docs/configuration/single-process-config.yaml 'go_gc_duration_seconds{job="prometheus"}'
 
 `


Following gets printed
 ```
 
 [{go_gc_duration_seconds __name__ 1} {go_gc_duration_seconds instance 1} {go_gc_duration_seconds job 1} {go_gc_duration_seconds quantile 2}]
result: error <nil> {__name__="go_gc_duration_seconds", instance="localhost:9090", job="prometheus", quantile="0"} => 0.0000235 @[1598798686000]
{__name__="go_gc_duration_seconds", instance="localhost:9090", job="prometheus", quantile="0.25"} => 0.0000359 @[1598798686000]
{__name__="go_gc_duration_seconds", instance="localhost:9090", job="prometheus", quantile="0.75"} => 0.0001225 @[1598798686000]
{__name__="go_gc_duration_seconds", instance="localhost:9090", job="prometheus", quantile="1"} => 0.0010739 @[1598798686000]

```

Above returns 4 labels with least no of unique values.    
The entire code is present here.

`https://github.com/jaybatra26/cortex/commit/eed05389d4c063455c68619c98385ab45fa5c925`

### Read and Write Path

* Implement selective indexing in the write path.
* Implement selective indexing in the read path (ie checking metadata and falling back to full scan)

The above part was a little tricky and involved a very deep understanding of the code. But my mentor [Bryan Boreham](https://twitter.com/bboreham) is an amazing guy and broke down complex parts into extremely simple form. 



Refer [1] for the list of commits over the past 3 months. 

### Extra Work Done:
* PR to change histogram buckets for metric `index_entries_per_chunk`: https://github.com/cortexproject/cortex/pull/3021
* Add validation check for config file: https://github.com/cortexproject/cortex/pull/2732

### Work to be done:
[PR](https://github.com/cortexproject/cortex/pull/2995) is to be merged(it is in mergeable state) and Documentation for the entire feature is left.


### Overall Learnings
* I learned the concept of standup. My mentor introduced me to a tool called as `standuply`. The idea is to familiarize with preoffesional development environment.
  This helped me organize my goals and track the progress as time passed.
* Setup of Cortex with DyanamoDB. I learned how to write docker-compose files and run a setup consisting of Prometheus, Cortex and DynamoDb.
* Deep understanding of how flushing of chunks takes place.
* Understanding of role of metrics.
* Understanding of how chunks look like and how are labels represented.
* Learned about maps of maps. It was a fairly complicated concepts for me as a beginner. I had a great learning in data processing here.
* Learned about how to create a type in golang and use list of maps.
* Learned about applying sorting on maps.
* Learned about what exactly are matchers in Prometheus. The concept of labels and matchers was little ambiguous in my mind.
* Learned how to write a design doc.(https://github.com/cortexproject/cortex/issues/2910)
* Learned how to process yaml in golang and how to test it. I got stuck with understanding how yaml works as tabs are not overlooked as spaces.
* Cortex uses inverted index, so if a query has a label, cortex would index it like<query_name>:<label_name>(go_gc_duration_seconds:instance). So if the prom query is like go_gc_duration_seconds{instance="localhost:9090"}, the value of labels inside curly braces is called as matchers in prometheus.
* Learned about testing metrics. With help from my mentor, I learned observability in tests which we used to test the metric values that change before and after indexing. The observations were as follows
#### Number of query lookups before excluding labels
    cortex_chunk_store_index_lookups_per_query_bucket{le="1"} 84
	cortex_chunk_store_index_lookups_per_query_bucket{le="2"} 120
	cortex_chunk_store_index_lookups_per_query_bucket{le="4"} 120
	cortex_chunk_store_index_lookups_per_query_bucket{le="8"} 120
	cortex_chunk_store_index_lookups_per_query_bucket{le="16"} 120
	cortex_chunk_store_index_lookups_per_query_bucket{le="+Inf"} 120
	cortex_chunk_store_index_lookups_per_query_sum 156
	cortex_chunk_store_index_lookups_per_query_count 120
 

#### Number of query lookups after excluding labels:

    cortex_chunk_store_index_lookups_per_query_bucket{le="1"} 120
	cortex_chunk_store_index_lookups_per_query_bucket{le="2"} 120
	cortex_chunk_store_index_lookups_per_query_bucket{le="4"} 120
	cortex_chunk_store_index_lookups_per_query_bucket{le="8"} 120
	cortex_chunk_store_index_lookups_per_query_bucket{le="16"} 120
	cortex_chunk_store_index_lookups_per_query_bucket{le="+Inf"} 120
	cortex_chunk_store_index_lookups_per_query_sum 120
	cortex_chunk_store_index_lookups_per_query_count 120

* Similarly, after implementing the Write Path, the number of index entries per chunk reduced as follows:
 #### Number of index entries before excluding labels:
        ` 
        cortex_chunk_store_index_entries_per_chunk_bucket{le="1"} 4
        cortex_chunk_store_index_entries_per_chunk_bucket{le="2"} 20
        cortex_chunk_store_index_entries_per_chunk_bucket{le="4"} 60
        cortex_chunk_store_index_entries_per_chunk_bucket{le="8"} 72
        cortex_chunk_store_index_entries_per_chunk_bucket{le="16"} 72
        cortex_chunk_store_index_entries_per_chunk_bucket{le="+Inf"} 72
        cortex_chunk_store_index_entries_per_chunk_sum 240
        cortex_chunk_store_index_entries_per_chunk_count 72
      
         
#### Number of index entries before excluding labels:
        `

            cortex_chunk_store_index_entries_per_chunk_bucket{le="1"} 4
            cortex_chunk_store_index_entries_per_chunk_bucket{le="2"} 20
            cortex_chunk_store_index_entries_per_chunk_bucket{le="4"} 64
            cortex_chunk_store_index_entries_per_chunk_bucket{le="8"} 72
            cortex_chunk_store_index_entries_per_chunk_bucket{le="16"} 72
            cortex_chunk_store_index_entries_per_chunk_bucket{le="+Inf"} 72
            cortex_chunk_store_index_entries_per_chunk_sum 232
            cortex_chunk_store_index_entries_per_chunk_count 72
        `

### Note

[1] Series of commits for Read and Write Path. Following is a new branch which is a copy of the original work which is stored in [storeconfig branch](https://github.com/cortexproject/cortex/pull/2995) and a PR is created.

