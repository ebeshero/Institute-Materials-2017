# eXist-db XQuery 3 and webapps

We continue with XQuery and get into how to create webapps in eXist-db.

## httpclient

`httpclient` provides functions for interacting with resources with a REST API. The following example connects to a server at Clarin and returns information about technical contact persons. It takes three parameters: the URL to process, a Boolean value that indicates whether the HTTP state (cookies, credentials, etc.) should persist for the life of the query (we set this to `False`), and optional request headers (in this case, the empty sequence). To construct the URL we append different numbers to a base URL of the Clarin REST interface <http://www.clarin.eu/restxml/> using the XPath `concat()` function, and then convert the string to a URI with the XPath `xs:anyURI` function. The `httpclient:get()` function returns a connection, to which we append the XPath path expressions `//cmd:TechnicalContact/cmd:Person` to return the information we need.

```xquery
xquery version "3.0";
declare namespace cmd="http://www.clarin.eu/cmd/";
<techContacts>{
	for $center-pos in ("1", "3", "4", "5", "6", "10", "11", "13", "20", "24", "25")
		(: (1 to 29) :)
	return
		httpclient:get(
    		xs:anyURI(concat("https://centres.clarin.eu/restxml/", $center-pos)),
    		false(),
    		())//cmd:TechnicalContact//cmd:Person
}</techContacts>
```
<!-- remove httpclient:post-form example because we didn’t use it
### httpclient example 2

```xquery
xquery version "3.0";

 httpclient:post-form(
 xs:anyURI("https://ws.example.com/wsauth/authenticate"), 
 <httpclient:fields>
    <httpclient:field name="username" value="{$username}" type="string"/>
    <httpclient:field name="password" value="{$password}" type="string"/>
    <httpclient:field name="checksum" value="{util:hash(concat($username, $passw
ord,
    $bas:config-doc//auth-salt), "md5")}" type="string"/>
 </httpclient:fields>,
 false(),
 <headers>
     <header name="Accept" value="text/xml"/>
 </headers>)
```
-->

## Index configuration

eXist-db constructs persistent indexes that support quick search and retrieval, much as a back-of-the-book index in a printed volume helps readers find specific content without having to look at every page. eXist-db will execute XQuery scripts with or without index support, but for unindexed queries will be slow. For professional-quality results, you must configure indexes. In addition to the [Configuring database indexes](https://exist-db.org/exist/apps/doc/indexing.xml), see the indexing section of [Tuning the database](http://exist-db.org/exist/apps/doc/tuning) for guidelines.

### Types of indexes

eXist-db constructs a structural index and an xml:id index for all XML documents automatically, but the other index types listed below have to be configured explicitly. 

* **structural:** element and attribute occurrences; created automatically
* **xml:id:** query with `fn:id()`; created automatically
* **range:** typed node values (used with general and value comparison and some functions, such as `contains()`). eXist-db has both a _new range index_ and a _legacy range index_. New projects should use the new range index.
* **ngram:** character substrings inside nodes (`ngram:contains()`)
* **Lucene:** full index of tokenized text. There is also a _legacy full-text index_, which should not be used.
* **Other:** geospatial, rdf-sparql, and others

### Location of configuration files

* Configuration files can be created separately for each collection and subcollection and must be placed in `/db/system/config/db/<path-to-collection>`
* Configuration files must end in `.xconf`, and are typically called `collection.xconf`.
* The collection hierarchy is mirrored in `/db/system/config/db/<path-to-collection>`
* Configurations on a lower level of the hierarchy overwrite configurations on higher levels

New documents are automatically indexed according to whatever indexes are in place when the documents are uploaded, but existing documents are not automatically reindexed when you update `collection.xconf`. To apply a revised configuration to existing documents, you must call `xmldb:reindex("/db/project/...")` in XQuery or use the Java admin client to reindex.

### Sample index configuration

The index configuration file is typically called `collection.xconf`, and you’ll need to read [Configuring database indexes](https://exist-db.org/exist/apps/doc/indexing.xml) to learn how to configure it. As a brief illustration, though, in the following example, we configure:

* The Lucene full-text index using the standard, whitespace, and keyword analyzers, and we tell it to index specific elements and attributes. You can read about Lucene analyzers at <https://exist-db.org/exist/apps/doc/lucene.xml>. 
* We create a new range index on the `@n` attribute. You can read about the new range index at <https://exist-db.org/exist/apps/doc/newrangeindex.xml>.
* We create an ngram index on `<head>` and `<p>` elements. You can read about the ngram index at <https://exist-db.org/exist/apps/doc/ngram>.

```xml
<collection xmlns="http://exist-db.org/collection-config/1.0">
    <index xmlns:tei="http://www.tei-c.org/ns/1.0" 
           xmlns:xs="http://www.w3.org/2001/XMLSchema-datatypes">
        <lucene>
            <analyzer class="org.apache.lucene.analysis.standard.StandardAnalyzer"/>
            <analyzer id="ws" class="org.apache.lucene.analysis.core.WhitespaceAnalyzer"/>
            <analyzer id="kw" class="org.apache.lucene.analysis.core.KeywordAnalyzer"/>
            <text qname="tei:w"/>
            <text qname="tei:head"/>
            <text qname="tei:p"/>
            <text qname="tei:lg"/>
            <text qname="tei:cell"/>
            <text qname="@atMost"/>
            <text field="age" qname="@atLeast"/>
            <ignore qname="tei:sic"/>
        </lucene>
        <range>
	        <create qname="@n" type="xs:string"/>
	    </range>
        <ngram qname="tei:head"/>
        <ngram qname="tei:p"/>
    </index>
</collection>
```

Namespace must be defined correctly.

### New range index

The new range index is used by:

* General and value comparison operators. The general comparison operators are `=`, `!=`, `<`, `<=`, `>`, `>=`. The value comparison operators are `eq`, `ne`, `lt`, `le`, `gt`, `ge`. The difference between general and value comparison is discussed at [What's the difference between "eq" and "="?](https://developer.marklogic.com/blog/comparison-operators-whats-the-difference).
* The functions `contains()`, `starts-with()`, and `ends-with()` (but not `matches()`).

The new range index requires you to specify a datatype for, e.g., `xs:integer`, `xs:decimal`, or other numerical types; `xs:string`; `xs:dateTime` and other date-time types, or `xs:boolean`. If you define an index with a specific type, all values in the indexed collections have to be valid instances of this type. For example, if you have indexed `<b>` elements as being of type `xs:integer`:

* `//a[b = 99]`: correct type, `xs:integer` index is used.
* `//a[b = "99"]`: wrong type, index not used (although you can force this in the main configuration file _conf.xml_)

### Lucene full-text index

The following example applies the Lucene full-text index to `<para>` elements anywhere and to `<section>` elements that are children of `<book>` elements. Within `<para>` elements, `<note>` descendants are not indexed and `<emphasis>` descendants are treated as part of their parents. (You need to specify “inline” in situations like `<word><prefix>un</prefix>clear</word>`. But default the Lucene index assumes that words end at element boundaries, but by specifying that `<prefix>` is inline, you can force the system to index “unclear” as a single word.

```xml
...
<lucene>
  <text qname=”para”>
    <ignore qname=”note”/>
    <inline qname=”emphasis”/>
  </text>
  <text match=”//book/section”/>
</lucene>
...
```

```xquery
ft:query($nodes as node()*, $query as item()) node()*
```
Requires a Lucene index on the collections in context.
The query expressed as either:
* Lucene query string, or
* an XML fragment defining the Lucene query

Term example:
```xquery
xquery version "3.1";

declare namespace tei = "http://www.tei-c.org/ns/1.0";

let $text := xmldb:xcollection("/db/dramawebben/data/works/StrindbergA_FrokenJulie")//tei:text
let $query := <query><term>mindre</term></query>
let $result := ft:query($text/descendant::tei:p, $query)
```
Wildcard example:
```xquery
$data-collection//tei:p[ft:query(.,<query><wildcard>ha*</wildcard></query>)]
```

Lucene query string example:
```
Wildcards: photograph*, photographer?
Multiple terms: native burmese
Phrase: “native burmese”~10
Fuzzy: photographer~
Required: +buddhist +burmese
Excluded: buddhist -burmese
Boost: buddhist^10 burmese
And: "caspian sea" AND tibet
Not: "caspian sea" NOT tibet
```
vs:
```xml
<bool>
  <term occur=”must”>caspian</term>
  <term occur=”should”>sea</term>
  <term occur=”not”>tibet</term>
</bool>
<near>caspian sea</near>
<phrase>caspian sea</phrase>
<bool>
  <near occur="must">caspian sea</near>
  <term occur="must">tibet</term>
</bool>
```

## Webapps
Recap of `typeswitch` function:
```xquery
xquery version "3.1";

declare namespace tei="http://www.tei-c.org/ns/1.0";

declare function local:gen-uuid($prefix as xs:string?) as xs:string {
concat($prefix, if (empty($prefix)) then "" else "-", util:uuid())
};


declare function local:add-uid-attributes-to-fragment($item as item())
 as item() {
  typeswitch($item)
    case document-node() return
        for $child in $item/node()
        return local:add-uid-attributes-to-fragment($child)
    case element() return
      element {node-name($item)} {
        $item/@*,
        if ($item[not(@uid)]) then attribute {"uid"} {local:gen-uuid(name($item))} else (),
        for $child in $item/node()
        return local:add-uid-attributes-to-fragment($child)
      }
    default return $item
};



let $data-collection := collection("/db/neh-2017")
let $result := "a string"
let $xml := <result><b val="b"> no bss</b> another word starting with a {$result}</result>

return local:add-uid-attributes-to-fragment($xml)
```

### Serialization
```xquery
declare option exist:serialize "method=html5 enforce-xhtml=yes";
```
* Controls how query output is sent to the browser
* Supported methods: `xml, xhtml, html5, text, json`
* `enforce-xhtml`: make sure all elements are in the XHTML default namespace

## Request
```xquery
request:get-parameter($name, $value)
```
`$name` is the name of the parameter and `$default`  is the default value to use if the parameter is not given.

See more functions in request and response modules:
* <http://demo.exist-db.org/exist/apps/fundocs/view.html?uri=http://exist-db.org/xquery/request&location=java:org.exist.xquery.functions.request.RequestModule>
* <http://demo.exist-db.org/exist/apps/fundocs/view.html?uri=http://exist-db.org/xquery/response&location=java:org.exist.xquery.functions.response.ResponseModule>


## Structure your app
_EXPath_ is an effort to standardize XQuery extension modules across implementations.
* Distribution of library modules (XQuery/Java/XSLT) as self-contained packages
* Register modules with the query engine
* Simple installation, hot deployment into running database

### eXist-db's application repository
* Extends EXPath packages
* Hot deployment and configuration of entire applications or services
* Self-contained, modular applications
* Foundation for component reuse
* Simplifies development process
* Simple deployment via eXist-db's [`dashboard`](http://localhost:8080/exist/apps/dashboard/index.html)

[`eXide`](http://localhost:8080/exist/apps/eXide/index.html) supports app development and generates app structure and puts your provided metadata in place. In eXide `Application->New application`.

Choose a name, fill in metadata, done. You will find it under `/db/apps/`. The collection structure is:
* `modules/` - (Everything XQuery)
* `templates/` - (Page templates used by html-files)
* `resources/` - (images, css, javascript) 
* `expath-pkg.xml` - (Package descriptor)
* `repo.xml` - (Application metadata)
* `index.html`
* `controller.xql` (URL rewriter)
* `pre-install.xql` - (executed before deployment)

Model-View-Controller concept:
* XML - model
* html - view
* XQuery - controller, implementing application logic

### HTML templating
```xml
<div class="templates:surround?with=templates/page.html&amp;at=content">
```
calls
```xquery
declare function templates:surround($node as node(), 
	$params as element(parameters)?, $model as item()*)
```

* Templating searches known modules for matching function
* Function signature is the same for all template functions

```xquery
declare function templates:surround($node as node(), 
	$params as element(parameters)?, $model as item()*)
```
Parameters:
* `$node`: HTML node which called the template
* `$params`: parameters passed
* `$model`: arbitrary data to be forwarded to nested templates

### URL Rewriting
Typical tasks:
* Check if a user is logged in and redirect to a login page
* Error handling
* Import resources from the file system
* Create REST-style URIs

