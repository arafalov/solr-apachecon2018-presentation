# Get the dataset
* Use _Data is Plural_ list of datasets as a dataset itself 
* https://docs.google.com/spreadsheets/d/1wZhPLMCHKJvwOkP4juclhjFgqIY8fQFMemwKL2c64vk/edit#gid=0
* Save as tsv (tab-separated values)

(The sequence below is similar but different from the one in the presentation)

# Setup Solr core
1. Start server in its own directory
    1. Create directory (e.g. _sroot_)
    1. Copy solr.xml into that directory from _\<solr install>/server/solr/_
    1. `bin/solr start -s .../sroot`
1.  Create core
    1. `bin/solr create -c dip -d _.../configsets/minimal_`
1. Try indexing downloaded tsv file
    1. Based on <https://lucene.apache.org/solr/guide/7_4/post-tool.html#indexing-csv>
            and <https://lucene.apache.org/solr/guide/7_4/uploading-data-with-index-handlers.html#csv-formatted-index-updates>
    1. `bin/post -c dip -params "separator=%09" -type "text/csv" .../dip-data.tsv`
    1. Fails as there is no _id_ field.
    1. Let's use rowid as unique id (`rowid=id`)
    1. `bin/post -c dip -params "separator=%09&rowid=id" -type "text/csv" .../dip-data.tsv`
1. Check in Solr Admin UI (Query screen)
    1. 605 records, seven fields (edition, position, headline, text, links, hattips, id)
    1. Search for *data* (in `q` field): 441 records
    1. Search for *DATA* - same, searches are case-insensitive
1. Let's quickly check how many entries we get by issue:
    1. <http://localhost:8983/solr/dip/select?facet.field=edition&facet=on&q=*:*&rows=0>
    1. Very consistent: 5 per issue
    1. What about by year ([with facet query](http://localhost:8983/solr/dip/select?facet=on&q=*:*&rows=0&facet.query={!prefix%20f=edition}2014&facet.query={!prefix%20f=edition}2015&facet.query={!prefix%20f=edition}2016&facet.query={!prefix%20f=edition}2017&facet.query={!prefix%20f=edition}2018)):
       ```
       http://localhost:8983/solr/dip/select?facet=on&q=*:*&rows=0
       &facet.query={!prefix%20f=edition}2014
       &facet.query={!prefix%20f=edition}2015
       &facet.query={!prefix%20f=edition}2016
       &facet.query={!prefix%20f=edition}2017
       &facet.query={!prefix%20f=edition}2018
       ```
    1. Ok, we can tell we really ramped-up in 2016 and then dropped in 2017.
       We'll see what this year is like.
# Iterate on schema design
## Make links multivalued, split on the space
1. First, let's check the definition ([Admin UI/Schema](http://localhost:8983/solr/#/dip/schema?field=links)), notice how it maps to dynamic field *
2. If we look in [the schema's definition file](http://localhost:8983/solr/#/dip/files?file=managed-schema), it is not there explicitly - that's the benefit of the current approach. Schemaless would be a different method as it detects types, but it has its own challenges that we (as commiters) are still trying to figure out. I like this one more.
3. So, let's create an explicit definition, possible via API/Admin UI because we are in a managed schema mode and are not locked-down.
4. In Admin UI, delete links and recreate them as stored/indexed/multivalued.
5. Normally, we could just reindex, but because it is docvalues and single->multivalued is a challenge - we need to delete the index first.
6. Command line delete: 
  
      ```bin/post -c dip -d $"<delete><query>*:*</query></delete>"```
7. Or can do it in the [Admin UI/Documents](http://localhost:8983/solr/#/dip/documents) screen, by putting this delete message as _Solr Command (raw XML or JSON)_ . Can also do it as JSON: 
    
    ```{"delete": { "query":"*:*" } }```
8. So, now we can index the TSV file again, and add instructions to split the _links_ field on space (by adding _f.links.split_ and _f.links.separator_ parameters)

```bin/post -c dip -params "separator=%09&rowid=id&f.links.split=true&f.links.separator=%20" -type "text/csv" .../dip-data.tsv```
    
9. Now we can rerun the basic query and see the links are split

## Find records with more than 3 links
1. Let's now find the records that have more than 3 links in one record. It is surprisingly hard because we are - effectively - searching dictionary, not the records themselves. So, "shape the data for search" - precalculate.
2. Introducing Update Request Processor - once the data format is normallized but before we hit the schema.
3. There is a lot of these URPs can do. If one can find them in the Reference Guide.... ([Well-Configured Solr Instance/Configuring solrconfig.xml/Update Request Processors](https://lucene.apache.org/solr/guide/7_4/update-request-processors.html))
4. We want _CountFieldValuesUpdateProcessorFactory_ but by itself it will replace a value; so we want to clone it first with _CloneFieldUpdateProcessorFactory_
5. We cannot do this in AdminUI yet, so we have two options:
    1. Update *solrconfig.xml* by hand (in non-SolrCloud setup) and reload the core;
    2. Use [Config API](https://lucene.apache.org/solr/guide/7_4/config-api.html) which creates an *configoverlay.json* file. Not everything that can be done by hand in *solrconfig.xml* can be done with Config API yet, but most things can be.
6. We are going to use *Config API*, in *Documents* screen, by setting *Request Handler* to "/config" and *Document Type* to "Solr Command":
    
```
{
    "add-updateprocessor": {
            "name": "cloneLinksToCount",
            "class": "solr.CloneFieldUpdateProcessorFactory",
            "source": "links",
            "dest":"linksCount"
    },
    "add-updateprocessor": {
        "name": "countLinks",
        "class": "solr.CountFieldValuesUpdateProcessorFactory",
        "fieldName": "linksCount"
    }
}
```

    Notice that we have a key-name duplication, that's one of the Solr's little extras to allow fit repeating constructs into JSON. It is not very good, but JSON is somewhat limited comparing to XML.

7. Let's check we see the definition in the [configoverlay.json](http://localhost:8983/solr/#/dip/files?file=configoverlay.json) file in the Files screen of the AdminUI
8. We are also going to explictly define linksCount field. We do not have to if we are ok with it as a string. But it would be nicer to actually recognize it is an integer, so we need to define the field first . 
    1. In Documents screen, set Request-Handler to */schema* and command:
    ```
    {"add-field-type" : {
            "name":"pint",
            "class":"solr.IntPointField",
            "docValues":"true"
    }}
    ```
    1. In Admin UI, create new field of int type with default set to 0. No need to delete records, we can just reindex.
9.  So, let's reindex, including the processor parameter that will invoke those URPs:

    ```
    bin/post -c dip -params "separator=%09&rowid=id&f.links.split=true&f.links.separator=%20&processor=cloneLinksToCount,countLinks" -type "text/csv" .../dip-data.tsv
    ```
10. And finally the query to find records with more than 3 links:
    ```http://localhost:8983/solr/dip/select?q=linksCount:[3 TO *]&sort=linksCount asc```

## Lock update parameters using Request Parameter API
1. POST to http://localhost:8983/solr/dip/config/params with the raw JSON body of:
```
{
  "set":{
    "DIP_INDEX":{
      "separator":"\t",
      "rowid":"id",
      "f.link.split": "true",
      "f.links.separator": " ",
      "processor":"cloneLinksToCount,countLinks"
}}}
```
1. The post command can now be:
```bin/post -c dip -params "useParams=DIP_INDEX" -type "text/csv" .../dip-data.tsv```

## Calculate average number of links (global and per per position)
1. Use [JSON Facet API](https://lucene.apache.org/solr/guide/7_4/json-facet-api.html)
1. Could use *json.facet* parameter, but it is a bit ugly for multi-lined json
1. Let's use POST body as a full request
### Average, min and max number of links, globally
1. POST to  http://localhost:8983/solr/dip/select with the raw JSON body of:
```
{
    "params":{
        "q":"*:*",
        "rows":5
    },
    "facet":{
        "avgLinks": "avg(linksCount)",
        "maxLinks": "max(linksCount)",
        "minLinks": "min(linksCount)"
    }
}
```
1. We should see 5 entries and then facets block with average of 3.4..., maximum of 13 and minimum of 1.
### Average and max number of links, broken down by the position
1. POST to the same URL with the raw JSON body of:
```
{
    params:{
        q:"*:*",
        rows:1
    },
    facet:{
        by_position: {
            type: terms,
            field: position,

            facet: {
                avgLinks: "avg(linksCount)",
                maxLinks: "max(linksCount)",
            }
        }
    }
}
```
2. We should get 1 record and then a facet breakdown by position and for each of them get avgLinks and maxLinks. Clearly, the lower (earlier) positions, have tendency for more links here.

# Let's find another dataset to explore
1. Our goal was to find another dataset, let's search for "cat"
1. We get some (4) "cat" but not all of them are about feline
1. Because we search  (defined in solrconfig.xml) \_text_ (notice underscores) and that's (in managed-schema) a copyfield of * to \_text_
1. Let's search against just headline and text, they are text_basic in dynamicField definition
   [http://localhost:8983/solr/dip/select?q=headline:cat text:cat]
1. That works, but is getting quite long, can we specify the fields we are looking at?
    1. Yes, as the query is processed by one (or multiple) query parsers and the 'df' is for the default one
    2. eDisMax gives us a lot more options, including fields to search with boosts
    ```
    defType=edismax
    &q=cat
    &qf=headline^10 text^5 _text_
    ```
    1. http://splainer.io/ (from OpenSourceConnections) is great to check what is going on there, can be run against your own local instance
1. So, that's eDisMax, we have another [25 or so specialized parsers](https://lucene.apache.org/solr/guide/7_4/other-parsers.html) which you can use standalone or in combination
1. What about __cats__ ?
    1. Gives a different result because we are not doing any language processing. Not because we cannot, because we do not know what you want.
    1. Compare _text_basic_ with (default schema's) _text_en_, _text_en_splitting_, _text_en_split_tight_
    1. And that's just English, Solr has many more types and you can compose your own, just have a look at
        1. _text_fa_ - demonstrating arabic and persian specific handling, and CharFilters
        1. My own definition that allows to search Thai with English phonetics, using bundled IBM's ICU4J library: [thai_english](https://github.com/arafalov/solr-thai-test/blob/master/collection1/conf/schema.xml#L34)
        1. My own definition to search phone number by suffix and associated presentation: [phone](https://github.com/arafalov/solr-presentation-2018-may/blob/master/configs/evolution/managed-schema#L33)
        1. The pipeline can be different for index and search, can have CharFilters, Tokenizer, and TokenFilters, see [my Solr Start resource website](http://www.solr-start.com/info/analyzers/)
    1. So, let's tell Solr that we know _text_ and _headline_ are English (_text_en_)
        1. Can do it via API, but - for non-Cloud instance - the config files are local and we can hack a bit and do copy/paste
            1. Copy field type definition
                1. from  _server/solr/configsets/\_default/conf/managed-schema_
                1. to _.../server/dip/conf/managed-schema_ (not original configset)
                1. anywhere in the file, because Solr does not care about the order in managed_schema (does in solrconfig.xml)
            1. Copy associated resource files as well:
                1. lang/stopwords_en.txt
                1. protwords.txt
                1. synonyms.txt
            1. Create explicit field definitions for _text_ and _headline_ using that field type
                ```<field name="text" type="text_en" indexed="true" stored="true"/>```
            1. Reload the core in Admin UI
            1. Reindex
        1. Rerun our searches (in splainer) and see that we now find plurals and non-plurals
        1. Also Admin UI's Analysis console allows to see how index and/or query pipeline deal with the text
1. So, let's pick a cute dataset to play with





# So far we learned
1. Getting Solr running, including with its own location
1. Creating new core, with custom configuration
1. Working with a TSV file, adding automatically-generated IDs
1. Modifying schema and configuration using API/Admin UI
1. Pre-processing records with Update Request Processors
1. Indexing, basic searching, and deleting records
1. Basic and advanced facets, including statistics

# Notes
1. Solr is an Index not a database, so much
1. Work backwards from search requirements
1. Be ready to reindex
1. Be ready to de-normalize, duplicate, and double-handle

