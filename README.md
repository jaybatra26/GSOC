# Selective Indexing of Labels

Selective indexing of labels:
The idea is to reduce the IO cost and storage cost by excluding some labels which are not very selective in nature. This would require adding a provision in the config file example a section named exclude_labels. While ingesting, this config is used to make sure that the ignored keywords are not ingested.

## Sub Tasks Included
1. Implement API endpoint to measure selectivity of labels in existing data.
2. Implement measurement of frequency of labels in queries.
3. Design a way to store and access metadata on indexing.
4. Design a way to measure the effectiveness of the finished project.

The above points required deep understanding of Cortex Service.
### Learnings:
* Setup of Cortex with DyanamoDB.
* Deep understanding of how flushing of chunks takes place.
* Understanding of role of metrics.
* Understanding of how chunks look like and how are labels represented.

    ### Implementation:
    The first task was to print the labels to identify how exactly how they look like? Initial implementation involved saving this into a CSV. A small script was written which fired all kinds of Prometheus queries called as [fire_queries.sh](https://github.com/jaybatra26/cortex/blob/leastlabel/fire_queries.sh)
    The next step was to count the least occurring labels. The code for the same goes into `pkg/chunk/composite_store.go`. 
    To get the information. First, Cortex is to be run, and `/flush` endpoint should be called. This has to be done because the default time of flush is 24hrs. 
    Once flushed, use Oneshot program(helper program to call single query).
    
    
     `./oneshot -as-at 2020-08-31T04:22:56Z  -config.file=./docs/configuration/single-process-config.yaml 'go_gc_duration_seconds{job="prometheus"}'`
     

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

    https://github.com/jaybatra26/cortex/commit/eed05389d4c063455c68619c98385ab45fa5c925

## Read and Write Path

* Implement selective indexing in the write path.
* Implement selective indexing in the read path (ie checking metadata and falling back to full scan)

The above part was a little tricky and involved a very deep understanding of the code. But my mentor [Bryan Boreham](https://twitter.com/bboreham?lang=en) is an amazing guy and broke down complex parts into extremely simple form. 

### Learnings:
Learned about observability in Prometheus. 
Used it to test read and write paths.

Refer [1] for the list of commits over the past 3 months. 

## Extra Work Done:
* PR to change histogram buckets for metric `index_entries_per_chunk`: https://github.com/cortexproject/cortex/pull/3021
* Add validation check for config file: https://github.com/cortexproject/cortex/pull/2732

## Work to be done:
Storeconfig [PR](https://github.com/cortexproject/cortex/pull/2995) is to be merged(it is in mergeable state) and Documentation for the entire feature is left.

### Note

[1] Series of commits for Read and Write Path. Following is a new branch which is a copy of the original work which is stored in [storeconfig branch](https://github.com/cortexproject/cortex/pull/2995) and a PR is created.

https://github.com/jaybatra26/cortex/commit/9922619ea45b557f0f763e56cb592503b6ec3bae
https://github.com/jaybatra26/cortex/commit/49fb029aeb31551a6846456d39ce492aff7c636c
https://github.com/jaybatra26/cortex/commit/cf5fde25f599930a1df1190d77e209eb9c9ba10f
https://github.com/jaybatra26/cortex/commit/e127161d47c3f8c7a1f5461447d3841049421bac
https://github.com/jaybatra26/cortex/commit/01c0170ea321153ee31b31cbec1899b753b81a33
https://github.com/jaybatra26/cortex/commit/1d4a53dc7fd94f9e167af84b6f9f37c1d611092d
https://github.com/jaybatra26/cortex/commit/cd9c43909e448c5f2454b0b4da8d4066e18fd628
https://github.com/jaybatra26/cortex/commit/03287949104162746adef95f551972220b783e14
https://github.com/jaybatra26/cortex/commit/0608b532c9615c1aeeee9952ba17116256a6d499
https://github.com/jaybatra26/cortex/commit/e7121d7b158449ec4da0737ad0106d89414c396a
https://github.com/jaybatra26/cortex/commit/74fdac5d8d8a02d08fe27464805660cc25744104
https://github.com/jaybatra26/cortex/commit/3d07aeb1b27040892354d9b8a16a5df727cfb8c2
https://github.com/jaybatra26/cortex/commit/b34725987f2728b193c09cebed65201d4f8876f2
https://github.com/jaybatra26/cortex/commit/e02f309305738d32fd6b20c4929e1b9757c9380d
https://github.com/jaybatra26/cortex/commit/83b3c43fc142b36d38187fbc84c5ba459e1a95d0
https://github.com/jaybatra26/cortex/commit/c304fe51cef9177fe1056d37035ce4340a5f914a
https://github.com/jaybatra26/cortex/commit/43583e32079044d866fadaca354464893c65327a
https://github.com/jaybatra26/cortex/commit/893afdd20e33d82cf176b162deaf575a3cf4bbd6
https://github.com/jaybatra26/cortex/commit/75f7d1905ebeb10b9d78768efd3cbdb9ced84f43
https://github.com/jaybatra26/cortex/commit/789011b6911de688c6d89666410da308db38cefe
https://github.com/jaybatra26/cortex/commit/789011b6911de688c6d89666410da308db38cefe
https://github.com/jaybatra26/cortex/commit/65049a902aec0b924481cbedd2709605a37942e3
https://github.com/jaybatra26/cortex/commit/ba72dbfcc016057e6267df01605616a53ddbc158
https://github.com/jaybatra26/cortex/commit/90b980bcfe3ace8ed77b5250bf6315bf0a9a9a57
https://github.com/jaybatra26/cortex/commit/dda9556d7764ae66a292e033aaf27d71eb4ac724
https://github.com/jaybatra26/cortex/commit/2ac68d9fccc103a89ba00753c5de39d52d95c0fb
https://github.com/jaybatra26/cortex/commit/57047ba57df239cef48ff21bfd42b8da4de3fcac
https://github.com/jaybatra26/cortex/commit/f9d7b3576ac5fbdc1c5bf28b49d02f7a6cb35d94
https://github.com/jaybatra26/cortex/commit/60d08572035b5a9cb1f049e1e86a4cdaf788a873
https://github.com/jaybatra26/cortex/commit/a48a4f7e0039b82b5451407bb19b0f5c9c60a21d
https://github.com/jaybatra26/cortex/commit/ba323b9dba52608e747c8ff7b2917dc81dd75683
https://github.com/jaybatra26/cortex/commit/3fb34e6c120529454fe3ca268172e3bfac30ead4
https://github.com/jaybatra26/cortex/commit/e28c8025f4e80580eaf294ea4f6367dbfa3cd92b
https://github.com/jaybatra26/cortex/commit/25abb8bd0f27280436de4867fa12fef1ca805907
https://github.com/jaybatra26/cortex/commit/444d7d9ce5c4731413ea594bed4f0b0cd490ba7d
https://github.com/jaybatra26/cortex/commit/77ebce147fa257fb7fc3e1602aa448318ba8623c
https://github.com/jaybatra26/cortex/commit/2c4b518fdfb192c14b25e3de3e21ea048d9f1cfa
https://github.com/jaybatra26/cortex/commit/b3887977f09e577dc979da84c43c90a76a0d8c0d
https://github.com/jaybatra26/cortex/commit/3c1a99a0f03b8bf68c045745923c71a43cdb1a0e
https://github.com/jaybatra26/cortex/commit/bdbf0e6a4042fcd3d3ea501cfd1cb80a3c11673a
https://github.com/jaybatra26/cortex/commit/963d1dbf97b9f27378daa7df708a07b6e02e4831
