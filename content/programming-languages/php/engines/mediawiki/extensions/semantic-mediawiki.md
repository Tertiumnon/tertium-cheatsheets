# Semantic MediaWiki

## Usage

### Semantic MediaWiki PHP examples

```php
<?php

// CREATE QUERY

// Simple way
$queryAsk = array("[[My property::Some text]]");
$res = SMWQueryProcessor::getResultFromFunctionParams( $queryAsk, SMW_OUTPUT_WIKI );

// [[Category:Person]]
$prop = new SMWDIWikiPage('Person', NS_CATEGORY);
$someProperty = new SMWClassDescription($prop, new SMWThingDescription);

// [[My property::+]]
$prop = new SMWDIProperty('My_property');
$someProperty = new SMWSomeProperty($prop, new SMWThingDescription);

// [[My property::Some text]]
$val = new SMWDIString('Some text');
$prop = new SMWDIProperty('My_property');
$valueDescription = new SMWValueDescription(
  $val,
  $prop
);
$someProperty = new SMWSomeProperty(
  $prop,
  $valueDescription
);

// GET RESULT

$query = new SMWQuery($someProperty);
$query->setLimit(10); // limit=10
$query->sort = true; // sorting
$query->sortkeys = array(
  '' => 'ASC'
); // by default property
$res = smwfGetStore()->getQueryResult($query);

?>
```

* [doc.semantic-mediawiki](https://doc.semantic-mediawiki.org)
* [group__SMWDataItems](https://doc.semantic-mediawiki.org/group__SMWDataItems.html)
* [2.2.x/includes/storage/SQLStore](https://github.com/SemanticMediaWiki/SemanticMediaWiki/tree/2.2.x/includes/storage/SQLStore)
* [Yury_Katkov/programming_examples](https://www.semantic-mediawiki.org/wiki/User:Yury_Katkov/programming_examples)
* [Yury_Katkov/programming_examples/Examples_not_working](https://www.semantic-mediawiki.org/wiki/Thread:User_talk:Yury_Katkov/programming_examples/Examples_not_working)
