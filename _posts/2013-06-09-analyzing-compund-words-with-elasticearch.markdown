---
layout: post
title:  "Analyzing compound words with elasticsearch"
date:   2013-06-09 09:17:30
categories: elasticsearch
---
Compound words are very common in Swedish and other european languages. In this post I will explain how you can use lucenes built in `TokenFilter` for analyzing them and making it posible to search for words that are apart of a compound word.

All the code for this example will be downloadable from <a href="http://github.com/pecke01/compound_example">this repo on GitHub</a>. You are more then welcome to download and use it.

First of all there are two recognized ways to split compound words. There are the statistcal approach that will use a statistical model for determining what and how to split a compound word. The mostly used approach is the one where you provide a list of words to work with. This is also by far the best one to use since it gives the best results.

In lucene one of the classes that performs compound splitting is named `DictionaryCompoundWordTokenFilter`. This is the one that I will first of all demonstrate how to use for you and then how we can extend it to work in different manner in another blog post. As the name points out it is dependant of a dictionary to work.

Since Swedish is my first language I will do the example in swedish. Should work the same way in any other languages.

<h2>Compound example in elasticsearch</h2>

In elasticsearch the lucene `DictionaryCompoundWordTokenFilter` maps directly to `dictionary_decompounder` which have can take a number of parameters. One of these are `word_list` which we will use in the example to provide a list of words. First we need to create a index that supports the use of this `TokenFilter`. I have a settings file that looks like this:
{%highlight json%}
{
    "settings" : {
        "index": {
            "number_of_shards": 1,
            "number_of_replicas": 0
        },
        "analysis": {
            "analyzer": {
                "marcusswedish" :{
                    "type" : "custom",
                    "tokenizer" : "standard",
                    "filter" : ["swedishTokenFilter"]
                }

            },
            "filter": {
                "swedishTokenFilter" : {
                    "type" : "dictionary_decompounder",
                    "word_list" : ["fotboll", "fot", "lag", "boll"],
                    "min_subword_size" : 2
                }
            }            
        }
    }
}
{%endhighlight%}
As you can see in the settings I have declared my own analyzer called marcusswedish which is a custom analyzer that uses the swedishTokenFilter I declared using the `dictionary_decompounder` that I mentioned previously.
To go with the settings I also have a mappings file that looks like this:
{%highlight json%}
{
    "mappings": {
        "_default_": {
            "_meta": {
                "type": "Compound word example"
            },
            "dynamic_templates": [
                {
                    "swedish": {
                        "mapping": {
                            "type": "string",
                            "analyzer": "marcusswedish",
                            "term_vector": "with_positions_offsets"
                        },
                        "match": "swedishstring"
                    }
                }
            ]
        }
    }
}
{%endhighlight%}

I created a very simple coffeescript that creates the index for us by merging the two files and calling elasticsearch running on localhost:9200. This is available in the <a href="http://github.com/pecke01/compound-example">repo on GitHub</a>.

{%highlight coffeescript%}
# Create an index with mappings.
console.log 'Creating index on localhost:9200'
indexname = 'compound_demo'
index = {}
# Settings
extend(true, index, JSON.parse(fs.readFileSync('settings_t.json')))
#Mappings
extend(true, index, JSON.parse(fs.readFileSync('mappings_t.json')))

client = restify.createJsonClient { url: 'http://localhost:9200/' }

#Delete index
client.del(indexname, (err, req, res, obj) ->
	)

client.post(indexname,index, (err ,req, res, obj) ->
	if err
		console.log 'Error when creating index: '+err
	else
		console.log  indexname+' created successfully'
	)
{%endhighlight%}

If I now index a document containing the word `fotbollslag` which is a compound of three words: `fot`, `boll` and `lag`

{%highlight bash%}
$ curl localhost:9200/compound_demo/type/1 -d '{"swedishstring":"Sveriges fotbollslag är i Österrike."}'
{%endhighlight%}

If we then try to search for the word fot in that field we should be able to get a hit that returns the indexed document. I have also turned on highlightning to show that partial highlightning will work.

{% highlight bash %}
$ curl -XGET 'localhost:9200/compound_demo/type/_search' -d '{
    "query": {
        "term": {
            "swedishstring": "fot"
        }
    },
    "highlight": {
        "fields": {
            "swedishstring": {}
        }
    }
 }'
{%endhighlight%}

We will the get the following response back and as you can see elasticsearch have highlighted fot of fotbollslag:
{%highlight json%}
{
  "took" : 6,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 0.13424811,
    "hits" : [ {
      "_index" : "compound_demo",
      "_type" : "type",
      "_id" : "1",
      "_score" : 0.13424811, "_source" : {"swedishstring":"Sveriges fotbollslag är i Österrike."},
      "highlight" : {
        "swedishstring" : [ "Sveriges <em>fot</em>bollslag är i Österrike." ]
      }
    } ]
  }
}
{%endhighlight%}

So we have now created support for compound word splitting in our index. I for one feels that the `DictionaryCompoundWordTokenFilter` is not that good at understanding swedish compund words. In my opinon `fotbollslag` should be split in to two words `fotboll`and `lag` since getting a match for fot isn't really what we would want in a great search application. In my next blog post I will show how we can extend the built in support to better work with swedish that have very few words that are compunded of several words where it would be useful to search for all of them.