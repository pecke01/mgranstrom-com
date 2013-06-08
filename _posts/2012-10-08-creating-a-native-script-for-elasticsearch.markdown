---
layout: post
title:  "Creating a native script for elasticsearch"
date:   2012-10-08 11:41:37
categories: elasticsearch
---
In this post I will give a step by step introduction to how you can create your own native scripts for ElasticSearch. This is targeted for `0.20.0` and might need some adjusting for later releases. 

By native in ES we mean Java. ES supports other languages as well but in this case I will show how to create a native script that is loaded on startup. In this dummy case I will make a native scripts that reverts a String field.

<h3>Step 1:</h3>

Implement a class that implement NativeScriptFactory. In this class override the method newScript. I am gonna return an instance of a class called Reverted. It will take the Map params as arguments for it's constructor.

{%highlight java%}
public class CustomScriptFactory implements NativeScriptFactory {
  @Override public ExecutableScript newScript (@Nullable Map params){
    return new Reverted(params);
  }
}
{% endhighlight %}

<h3>Step 2:</h3>

Implement the actual class that runs the script. In this case Reverted. I will give you an example of this class as well. It actually doesn't do a lot except reverts a field and returns it. My reverted class will extend the AbstractSearchScript and override the run method. It will also just get a substring of the reverted string to show that you can have several parameters to a script.

{%highlight java%}
public class Reverted extends AbstractSearchScript {
  String fieldParam;
  int lengthParam;
  public Reverted(@Nullable Map params){
    fieldParam = (String)params.get("field");
    lengthParam = new Integer(params.get("length").toString()).intValue();
  }
  @Override
  public Object run(){
    if(doc().containsKey(fieldParam) && doc().field(fieldParam) != null && doc().field(fieldParam).getStringValue() != null) { 
      String field = doc().field(fieldParam).getStringValue();
      String reverse = new StringBuffer(field).reverse().toString();

      return reverse.substring(0, lengthParam); 
    }
    else {           
      return "";
    }
  }       
}
{% endhighlight %}

<h3>Step 3:</h3>

Build and pack it into a JAR File that can be loaded.

{%highlight bash%}
$ javac -cp ..\elasticsearch-0.19.8\lib\elasticsearch-0.19.8.jar .\com\marcus\elasticsearch\scripts\*.java
$ jar cf marcus-scripts-1.0.jar .\com\marcus\elasticsearch\scripts\*.class
{% endhighlight %}

<h3>Step 4:</h3>

Add config row to elasticsearch.yml.

{%highlight bash%}
script.native: revert.type: com.marcus.elasticsearch.scripts.CustomScriptFactory
{% endhighlight %}

<h3>Step 5:</h3>

Now we are actually done. I place the created jar in the lib folder of my local ES and start it. We can now see if it works like we intended it to do. I index a document, the famous example from the ElasticSearch webpage.

{%highlight bash%}
$ curl -XPUT 'http://localhost:9200/twitter/tweet/1' -d '{
    "user" : "kimchy",
    "post_date" : "2009-11-15T14:12:12",
    "message" : "trying out Elastic Search"
}'
{% endhighlight %}

After I have indexed a document I query to see if the script works by calling the native script and reverting the message field.

{%highlight bash%}
$ curl -XPUT 'http://localhost:9200/twitter/_search' -d '{
"fields" : ["user","message"],
 "query" : {
        "term" : { "user" : "kimchy" }
    },
    "script_fields": {
        "MessageReverted": {
            "script": "revert",
            "lang": "native",
            "params": {
                "field": "message",
                "length": 6
            }
        }
    }
}'
{% endhighlight %}
RESULT:
{%highlight json%}
{
    "took": 6,
    "timed_out": false,
    "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
    },
    "hits": {
        "total": 1,
        "max_score": 0.30685282,
        "hits": [{
            "_index": "twitter",
            "_type": "tweet",
            "_id": "1",
            "_score": 0.30685282,
            "fields": {
                "message": "trying out Elastic Search",
                "user": "kimchy",
                "MessageReverted": "hcraeS"
            }
        }]
    }
}
{% endhighlight %}

As you can see the result is as we intended it to be. Hope this will help you out when creating a native script for ElasticSearch. We can do a lot with native scripts and remember that they will almost always be faster then sending in a script. It will also reduce bandwidth for you.