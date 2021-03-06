---
layout: default
---

# Example Application: A Mention-Level Extraction System

This document describes how to **build an application to extract mention-level
`has_spouse` relation from text** in DeepDive. 

This document assumes you are familiar with basic concepts in DeepDive and in
[Knowledge Base Construction](../../general/kbc.html). Please refer to other
[documents](../../../index.html#documentation) to learn more.

## <a name="high_level_picture" href="#"> </a> High-level picture of the application

This tutorial shows how to build a full DeepDive application that extracts
**mention-level** `has_spouse` relationships from raw text. We use news articles
as the input data and we want to extract all pairs of people that participate in
a `has_spouse` relation, for example *Barack Obama* and *Michelle Obama*. This
example can be easily translated into relation extraction in other domains, such
as extracting interactions between drugs, or relationships among companies.

Note that in this section we are going to build an application that extract
*plausible* mention-level relations, but *with rather low quality*.

<a name="dataflow" href="#"> </a>

On a high level, the application will perform following steps on the data:

1. Data preprocessing: prepare parsed sentences.
2. Feature extraction: 
  - Extract mentions of people in the text
  - Extract all candidate pairs of people that possibly participate in a `has_spouse` relation
  - Extract features for `has_spouse` candidates
  - Prepare training data by [distant supervision](../../general/relation_extraction.html) using a knowledge base
3. Generate factor graphs by inference rules
4. statistical inference and learning
5. Generate results

The full application is also available in the folder
`DEEPDIVE_HOME/examples/spouse_example`, which contains all possible
implementations in [different types of extractors](../extractors.html). In this
document, we only introduce the default extractor type (`json_extractor`), which
correspond to `DEEPDIVE_HOME/examples/spouse_example/default_extractor` in your
DeepDive directory.

This tutorial assumes you [installed DeepDive](../installation.html) and denotes
the `deepdive` installation directory as `$DEEPDIVE_HOME`. We will be using
PostgreSQL as our primary database in this example. If you followed the DeepDive
installation guide and passed all tests then your PostgreSQL server should
already be running.

### Contents

* [Preparation](#preparation):
* [Implement the data flow](#implement_dataflow):
  1. [Data preprocessing](#loading_data)
  2. [Feature extraction](#feature_extraction)
      - [Adding a people extractor](#people_extractor)
      - [Extracting candidate relations](#candidate_relations)
      - [Adding features for candidate relations](#candidate_relation_features)
  3. [Writing inference rules for factor graph generation](#inference_rules)
  4. [Statistical inference and learning](#learning_inference)
  5. [Get and check results](#get_result)

Other sections:

- [How to examine / improve results](walkthrough-improve.html)
- [Extras: preprocessing, NLP, pipelines, debugging
  extractor](walkthrough-extras.html)
- [Go back to the Walkthrough](walkthrough.html)

## <a name="preparation" href="#"></a> Preparation

Start by creating a new application folder `app/spouse` in the DeepDive
directory:

```bash
cd $DEEPDIVE_HOME
mkdir -p app/spouse   # make folders recursively
cd app/spouse
```

DeepDive's main entry point is a file called `application.conf` which contains
database connection information as well as your feature extraction and inference
rule pipelines. It is often useful to have a `env.sh` script to configure
environmental variable and a `run.sh` script that loads `env.sh` and executes
the DeepDive pipeline. We provide simple templates for these files to copy and
modify. Copy these templates to the application directory: 

```bash
cp ../../examples/template/application.conf .
cp ../../examples/template/run.sh .
cp ../../examples/template/env.sh .
```

The `env.sh` file contains a placeholder line `DBNAME=` in that file, modify
`env.sh` and fill it with your database name:
  
```bash
# File: env.sh
...
export DBNAME=deepdive_spouse   # modify this line
...
```

Note that in the `run.sh` file it defines environment variables `APP_HOME` which
is our current directory `app/spouse`, and `DEEPDIVE_HOME` which is the home
directory of DeepDive. We will use these variables later.

You can now try running the `run.sh` script:

```bash
./run.sh
```

Since no [extractors](../extractors.html) or [inference
rules](../inference_rules.html) have been specified, the results are not
meaningful, but DeepDive should run successfully from end to end and you should
be able to see a summary report such as:

    15:57:55 [profiler] INFO  --------------------------------------------------
    15:57:55 [profiler] INFO  Summary Report
    15:57:55 [profiler] INFO  --------------------------------------------------

## <a name="implement_dataflow" href="#"></a> Implement the Data Flow

Now let's start implementing the [data flow](#dataflow) for this KBC application.

### <a name="loading_data" href="#"></a> Data preprocessing and loading

For simplicity, we will start from preprocessed sentence data (starting
from step 2). If the input was raw text articles, we also need to run
natural language processing in order to extract candidate pairs and
features. If you want to learn how NLP extraction can be done in
DeepDive (starting from step 1), you can refer to the
[appendix](walkthrough-extras.html#nlp_extractor).

Copy the prepared data and scripts from the
`DEEPDIVE_HOME/example/spouse_example/` folder into `DEEPDIVE_HOME/app/spouse/`:

```bash
cp -r ../../examples/spouse_example/data .
cp ../../examples/spouse_example/schema.sql .
cp ../../examples/spouse_example/setup_database.sh .
```

Then, execute the script `setup_database.sh`, which create the database and the
necessary table and load the preprocessed data.

```bash
sh setup_database.sh deepdive_spouse
```

The following relations are created:

                         List of relations
     Schema |        Name         | Type  |  Owner   | Storage
    --------+---------------------+-------+----------+---------
     public | articles            | table | deepdive | heap
     public | has_spouse          | table | deepdive | heap
     public | has_spouse_features | table | deepdive | heap
     public | people_mentions     | table | deepdive | heap
     public | sentences           | table | deepdive | heap
    (5 rows)

Among these relations: 

- `articles` contains article data
- `sentences` contains processed sentence data by an [NLP
  extractor](walkthrough-extras.html#nlp_extractor). This table contains
  tokenized words, lemmatized words, POS tags, NER tags, dependency paths for
  each sentence.
- the other tables are currently empty, and will be populated during the feature
  extraction step.

For a detailed reference of how these tables are created and loaded, refer to
[Preparing Data Tables](walkthrough-extras.html#data_tables) in the extra
section.

### <a name="feature_extraction" href="#"></a> Feature Extraction

Our next task is to write several [extractors](../extractors.html) for feature extraction. 

#### <a name="people_extractor" href="#"></a> Adding a people extractor

We first need to recognize the persons mentioned in the article. Given a
set of sentences, the person mention extractor will populate a relation
`people_mention` that contains an encoding of the mentions. In this example, we
build a simple person recognizer that just uses the NER tags from the underlying
NLP toolkit. We could do something more sophisticated, but we just want to
illustrate the basic concepts.

**Input:** sentences along with NER tags. Specically, each line in the input to
this extractor UDF is a row in `sentence` table in JSON format, e.g.:

    {"sentence_id":"118238@10","words":["Sen.","Barack","Obama","and","his","wife",",","Michelle","Obama",",","have","released","eight","years","of","joint","returns","."],"ner_tags":["O","PERSON","PERSON","O","O","O","O","PERSON","PERSON","O","O","O","DURATION","DURATION","O","O","O","O"]}

**Output:** rows in `people_mentions` table, e.g.:

    {"mention_id": "118238@10_1", "text": "Barack Obama", "sentence_id": "118238@10", "start_position": 1, "length": 2}
    {"mention_id": "118238@10_7", "text": "Michelle Obama", "sentence_id": "118238@10", "start_position": 7, "length": 2}


This first extractor extracts people mentions (In the sample data, "Barack
Obama" and "Michelle Obama") from the sentences, and put them into a new table. 
Note that we have named entity tags in column `ner_tags` of our `sentences`
table. We will use this column to identify people mentions: we assume that *a
word phrase is a people mention if all its words are tagged as `PERSON` in its
`ner_tags` field.*

<!-- Ideally you would want to add your own domain-specific features to extract
mentions. For example, people names are usually capitalized, tagged with a noun
phrase part of speech tag, and have certain dependency paths to other words in
the sentence. However, because the Stanford NLP Parser is relatively good at
identifying people and tags them with a `PERSON` named-entity tag we trust its
output and don't make the predictions ourselves. We simply assume that all
people identified by the NLP Parser are correct. Note that this assumption is
not ideal and usually does not work for other types of entities, but it is good
enough to build a first version of our application.
 -->

To define our extractors in DeepDive, we start by adding several lines
into the `deepdive.extraction.extractors` block in `application.conf`, which
should be present in the template:

```bash
deepdive {
  ...
  # Put your extractors here
  extraction.extractors {

    # Extractor 1: Clean output tables of all extractors
    ext_clear_table {
      style: "sql_extractor"
      sql: """
        DELETE FROM people_mentions;
        DELETE FROM has_spouse;
        DELETE FROM has_spouse_features;
        """
    }
    
    # Extractor 2: extract people mentions:
    ext_people {
      # An input to the extractor is a row (tuple) of the following query:
      input: """
        SELECT  sentence_id, words, ner_tags
          FROM  sentences"""

      # output of extractor will be written to this table:
      output_relation: "people_mentions"

      # This user-defined function will be performed on each row (tuple) of input query:
      udf: ${APP_HOME}"/udf/ext_people.py"

      dependencies: ["ext_clear_table"]
    }
    # ... (more extractors to add here)
  } 
  ...   
}
```

Note that we first create an extractor `ext_clear_table` to be executed
before any other extractor, to clear output tables of all other
extractors. This is a `sql_extractor`, which is just a set of SQL
commands specified in the `sql` field.

For our person mention extractor `ext_people`, the meaning of each line is the
following:

1. The input to the `ext_people` extractor are all sentences (identifiers, words
and named entity tags), selected using a SQL statement.

2. The output of the extractor will be written to the `people_mentions` table.
(for the table format, refer to the [cheat-sheet](#table_cheatsheet) in
appendix.)

3. The extractor script is `udf/ext_people.py`. DeepDive will execute this
command and stream database tuples to the *stdin* of the process, and read output from
*stdout* of the process.

4. The `dependencies` field specifies that this extractor can be executed only
after `ext_clear_table` extractor finishes.

For additional information about extractors, refer to the ['Writing extractors'
guide](../extractors.html).

Create now a `udf` directory to store the scripts:

```bash
mkdir udf
```

Then create a `udf/ext_people.py` script as the UDF for this extractor, which
scans through input sentences and output phrases. The script contains the
following code:

```python
#! /usr/bin/env python
# File: udf/ext_people.py

import json, sys

# For-loop for each row in the input query
for row in sys.stdin:
  # load JSON format of the current tuple
  sentence_obj = json.loads(row)
  # Find phrases that are continuous words tagged with PERSON.
  phrases = []   # Store (start_position, length, text)
  words = sentence_obj["words"]         # a list of all words
  ner_tags = sentence_obj["ner_tags"]   # ner_tags for each word
  start_index = 0
  
  while start_index < len(words):
    # Checking if there is a PERSON phrase starting from start_index
    index = start_index
    while index < len(words) and ner_tags[index] == "PERSON": 
      index += 1
    if index != start_index:   # found a person from "start_index" to "index"
      text = ' '.join(words[start_index:index])
      length = index - start_index
      phrases.append((start_index, length, text))
    start_index = index + 1
  
  # Output a tuple for each PERSON phrase
  for start_position, length, text in phrases:
    print json.dumps({
      "sentence_id": sentence_obj["sentence_id"],
      "start_position": start_position,
      "length": length,
      "text": text,
      # Create an unique mention_id by sentence_id + offset:
      "mention_id": '%s_%d' % (sentence_obj["sentence_id"], start_position)
    })
```

If you want to get a sample of the inputs to the extractor, refer to [getting
example inputs](walkthrough-extras.html#debug_extractors) section in the Extras.

This `udf/ext_people.py` Python script takes sentences records as an input, and
outputs a people record for each (potentially multi-word) person phrase found in
the sentence, which are one or multiple continuous words tagged with `PERSON`.

Note that if you wanted to add debug output, you can print to *stderr* instead
of stdout, and the messages would appear on the terminal, as well as in the log
file (`$DEEPDIVE_HOME/log/DATE_TIME.txt`).

You can now run your extractor by executing `./run.sh`, and you will be able to
see extracted results in the `people_mentions` table. We select the results of
the sample input data to see what happens in the extractor.

```bash
./run.sh    # Run extractors now
psql -d deepdive_spouse -c "select * from people_mentions where sentence_name='118238@10'"
```

The results are:

     sentence_id | start_position | length |      text      | mention_id
    -------------+----------------+--------+----------------+-------------
     118238@10   |              1 |      2 | Barack Obama   | 118238@10_1
     118238@10   |              7 |      2 | Michelle Obama | 118238@10_7
    (2 rows)

To double-check that the results you obtained are correct, count the number of
tuples in your tables:

```bash
psql -d deepdive_spouse -c "select count(*) from sentences;"
psql -d deepdive_spouse -c "select count(*) from people_mentions;"
```

The results should be:

     count
    -------
     43789

     count
    -------
     39266

#### <a name="candidate_relations" href="#"></a> Extracting candidate relations between mention pairs

Now we need to identify candidates for the `has_spouse` relation. For
simplicity, we only extract each **pair of person mentions in the same
sentence** as a relation candidate. 

To train our system to decide whether these candidates are correct relations, we
also need to generate some training data. However, it is hard to find ground
truth on whether two mentions participate in `has_spouse` relation. We use
[distant supervision](../../general/distant_supervision.html) that generates
mention-level training data by heuristically mapping them to known entity-level
relations in an existing knowledge base.

This extractor will take all mentions of people from a same sentence, and put
each pair of them into the table `has_spouse`, while generating true / false
labels on some of the mention pairs.

**Input:** two mentions from the `people_mentions` table coming from a same sentence. e.g.:

    {"p1_text":"Michelle Obama","p1_mention_id":"118238@10_7","p2_text":"Barack Obama","p2_mention_id":"118238@10_1","sentence_id":"118238@10"}

**Output:** one row in `has_spouse` table:

    {"person1_id": "118238@10_7", "person2_id": "118238@10_1", "sentence_id": "118238@10", "relation_id": "118238@10_7_118238@10_1", "description": "Michelle Obama-Barack Obama", "is_true": true, "id": null}

To understand how DeepDive works, we should notice that the `has_spouse` table
has following format:

         Table "public.has_spouse"
       Column    |  Type   | Modifiers
    -------------+---------+-----------
     person1_id  | bigint  |            # first person's mention_id in people_mentions
     person2_id  | bigint  |            # second person's mention_id
     sentence_id | bigint  |            # which senence it appears
     description | text    |            # a description of this relation pair
     is_true     | boolean |            # whether this relation is correct
     relation_id | bigint  |            # unique identifier for has_spouse
     id          | bigint  |            # reserved for DeepDive


Note that in table `has_spouse`, there is a special `is_true` column. We need
this column because we want DeepDive to predict how likely it is that a given
entry in the table is correct. Each row in the `has_spouse` table will be
assigned a variable for its `is_true` column, and DeepDive will predict
the value for this variable.

Also note that we must reserve another special column, `id`, of type `bigint`,
in any table containing variables like this one. This column should be **left
blank, and not be used anywhere**. We will further see syntax requirements in
*inference rules* related to this `id` column.  

We now tell DeepDive to create variables for the `is_true` column of the
`has_spouse` table for probabilistic inference, by adding it into the
`schema.variables` block in `application.conf`:

    schema.variables {
      has_spouse.is_true: Boolean
    }

We now create an extractor that extracts all candidate relations and puts them
into the above table. We call them *candidate relations* because we are not sure
whether or not they are actually correct: that's for DeepDive to predict. Add
the following to `application.conf`:

```bash
extraction.extractors {

  # ... (other extractors)

  # Extractor 3: extract mention relation candidates
  ext_has_spouse_candidates {
    # Each input (p1, p2) is a pair of mentions
    input: """
      SELECT  sentences.sentence_id,
              p1.mention_id AS p1_mention_id,
              p1.text       AS p1_text,
              p2.mention_id AS p2_mention_id,
              p2.text       AS p2_text
       FROM   people_mentions p1,
              people_mentions p2,
              sentences
      WHERE   p1.sentence_id = p2.sentence_id
        AND   p1.sentence_id = sentences.sentence_id
        AND   p1.mention_id != p2.mention_id;
        """
    output_relation : "has_spouse"
    udf             : ${APP_HOME}"/udf/ext_has_spouse.py"

    # Run this extractor after "ext_people"
    dependencies    : ["ext_people"]
  }

}
```

Note that this extractor must be executed after our previously added extractor `ext_people`,
which is specified in "dependencies" field.

When generating relation candidates, we also generate training data using
[distant supervision](../../general/distant_supervision.html). There are some pairs of
people that we know for sure are married, and we can use them as training data
for DeepDive. Similarly, if we know that two people are not married, we can use
them as negative training examples. In our case we will be using data from
[Freebase](http://www.freebase.com/) for distant supervision, and use direct
string match to map mentions to entities.

To generate positive examples, we have exported all pairs of people with a
`has_spouse` relationship from the [Freebase data
dump](https://developers.google.com/freebase/data) and included them in a CSV
file `data/spouses.csv`.

To generate negative examples, we use two heuristics:

1. A pair of the same person is a negative example of `has_spouse` relations,
e.g., "Barack Obama" cannot be married to "Barack Obama". 

2. A pairs of some relations incompatible with `has_spouse` can be treated as a
negative example if, for example, A is B's parent / children / sibling, or A is
not likely to be married to B.  We include a TSV file in `data/non-spouses.tsv`
containing such relations sampled from Freebase.

We now create a script `udf/ext_has_spouse.py` to generate and label
the relation candidates:

```python
#! /usr/bin/env python
# File: udf/ext_has_spouse.py

import json, csv, os, sys
BASE_DIR = os.path.dirname(os.path.realpath(__file__))

# Load the spouse dictionary for distant supervision
spouses = set()  # One person may have multiple spouses, so use set
with open (BASE_DIR + "/../data/spouses.csv") as csvfile:
  reader = csv.reader(csvfile)
  for line in reader:
    name1 = line[0].strip().lower()
    name2 = line[1].strip().lower()
    spouses.add((name1, name2))
    spouses.add((name2, name1))

# Load relations of people that are not spouse
non_spouses = set()
lines = open(BASE_DIR + '/../data/non-spouses.tsv').readlines()
for line in lines:
  name1, name2, relation = line.strip().split('\t')
  non_spouses.add((name1, name2))  # Add a non-spouse relation pair
  non_spouses.add((name2, name1))

# For each input tuple
for row in sys.stdin:
  obj = json.loads(row)
  p1_text = obj["p1_text"].strip()
  p2_text = obj["p2_text"].strip()
  p1_lower = p1_text.lower()
  p2_lower = p2_text.lower()

  is_true = None
  if (p1_lower, p2_lower) in spouses:
    is_true = True    # the mention pair is in our supervision dictionary
  elif (p1_lower, p2_lower) in non_spouses: 
    is_true = False   # they appear in other relations
  elif (p1_text == p2_text) or (p1_text in p2_text) or (p2_text in p1_text):
    is_true = False   # they are the same person

  # Create an unique relation_id by a combination of two mention's IDs
  relation_id = obj["p1_mention_id"] + '_' + obj["p2_mention_id"]

  print json.dumps({"person1_id": obj["p1_mention_id"],  "person2_id": obj["p2_mention_id"],
    "sentence_id": obj["sentence_id"],  "description": "%s-%s" %(p1_text, p2_text),
    "is_true": is_true,  "relation_id": relation_id,  "id": None
  })
```


Now if you like, you can run the system by executing `run.sh` and you can check
the output relation `has_spouse`. `run.sh` will run the full pipeline with both
extractors `ext_people` and `ext_has_spouse`. If you only want to run your new
extractor, refer to the [Pipeline section in Extras](walkthrough-extras.html#pipelines).

```bash
./run.sh
psql -d deepdive_spouse -c "select * from has_spouse where person1_name='118238@10_7'"
```

Results look like:

     person1_id  | person2_id  | sentence_id |         description         | is_true |       relation_id       | id
    -------------+-------------+-------------+-----------------------------+---------+-------------------------+----
     118238@10_1 | 118238@10_7 | 118238@10   | Barack Obama-Michelle Obama | t       | 118238@10_1_118238@10_7 |


To check that your results are correct, you can count the number of tuples in
the table:

```bash
psql -d deepdive_spouse -c "select is_true, count(*) from has_spouse group by is_true;"
```

The results should be:

     is_true | count
    ---------+-------
     f       |  3770
     t       |  2234
             | 69418

#### <a name="candidate_relation_features" href="#"></a> Adding Features for candidate relations

In order for DeepDive to make predictions, we need to add *features* to our
candidate relations. Features are properties that help decide whether or not the
given relation is correct. We use standard machine learning features, but this
will result in low quality which we will improve later. We will write an
extractor that extracts features from mention pairs and the sentences they come
from.

The features we use are: (1) the bag of words between the two mentions;
(2) the number of words between two phases; (3) whether the last word of two
person's names (last name) is the same.
We will refine these features [later](walkthrough-improve.html).

For this new extractor:

**Input:** a mention pair as well as all words in the sentence it appears. e.g.:

    {"p2_length":2,"p1_length":2,"words":["Sen.","Barack","Obama","and","his","wife",",","Michelle","Obama",",","have","released","eight","years","of","joint","returns","."],"relation_id":"118238@10_1_118238@10_7","p1_start":1,"p2_start":7}

**Output:** all features for this mention pair described above:

    {"relation_id": "118238@10_1_118238@10_7", "feature": "word_between=,"}
    {"relation_id": "118238@10_1_118238@10_7", "feature": "word_between=his"}
    {"relation_id": "118238@10_1_118238@10_7", "feature": "potential_last_name_match"}
    {"relation_id": "118238@10_1_118238@10_7", "feature": "word_between=wife"}
    {"relation_id": "118238@10_1_118238@10_7", "feature": "num_words_between=4"}
    {"relation_id": "118238@10_1_118238@10_7", "feature": "word_between=and"}

Create a new extractor for features, which will execute after the
`ext_has_spouse_candidates` extractor:

```bash
extraction.extractors {

  # ... (other extractors)

  # Extractor 4: extract features for relation candidates
  ext_has_spouse_features {
    input: """
      SELECT  sentences.words,
              has_spouse.relation_id,
              p1.start_position  AS  p1_start,
              p1.length          AS  p1_length,
              p2.start_position  AS  p2_start,
              p2.length          AS  p2_length
        FROM  has_spouse,
              people_mentions p1,
              people_mentions p2,
              sentences
       WHERE  has_spouse.person1_id = p1.mention_id
         AND  has_spouse.person2_id = p2.mention_id
         AND  has_spouse.sentence_id = sentences.sentence_id;
         """
    output_relation : "has_spouse_features"
    udf             : ${APP_HOME}"/udf/ext_has_spouse_features.py"
    dependencies    : ["ext_has_spouse_candidates"]
  }

}
```

To create our extractor UDF, we will make use of `ddlib`, our Python library
that provides useful utilities such as `Span` to manipulate elements in
sentences. Make sure you followed the [installation
guide](../installation.html#ddlib) to properly use `ddlib`.

Create the script `udf/ext_has_spouse_features.py` as follows:

```python
#! /usr/bin/env python
# File: udf/ext_has_spouse_features.py

import sys, json
import ddlib     # DeepDive python utility

# For each input tuple
for row in sys.stdin:
  obj = json.loads(row)
  words = obj["words"]
  # Unpack input into tuples.
  span1 = ddlib.Span(begin_word_name=obj['p1_start'], length=obj['p1_length'])
  span2 = ddlib.Span(begin_word_name=obj['p2_start'], length=obj['p2_length'])

  # Features for this pair come in here
  features = set()
  
  # Feature 1: Bag of words between the two phrases
  words_between = ddlib.tokens_between_spans(words, span1, span2)
  for word in words_between.elements:
    features.add("word_between=" + word)

  # Feature 2: Number of words between the two phrases
  features.add("num_words_between=%s" % len(words_between.elements))

  # Feature 3: Does the last word (last name) match?
  last_word_left = ddlib.materialize_span(words, span1)[-1]
  last_word_right = ddlib.materialize_span(words, span2)[-1]
  if (last_word_left == last_word_right):
    features.add("potential_last_name_match")

  # # Use this line if you want to print out all features extracted:
  # ddlib.log(features)

  for feature in features:  
    print json.dumps({
      "relation_id": obj["relation_id"],
      "feature": feature
    })
```

As before, you can run the system by executing `run.sh` and check the output
relation `has_spouse_features`. (If you only want to run your new extractor,
refer to the [Pipeline section in Extras](walkthrough-extras.html#pipelines).)

```bash
./run.sh
psql -d deepdive_spouse -c "select * from has_spouse_features where relation_id = '118238@10_1_118238@10_7'"
```

The results look like:

```
       relation_id       |          feature
-------------------------+---------------------------
 118238@10_1_118238@10_7 | word_between=,
 118238@10_1_118238@10_7 | word_between=his
 118238@10_1_118238@10_7 | potential_last_name_match
 118238@10_1_118238@10_7 | word_between=wife
 118238@10_1_118238@10_7 | num_words_between=4
 118238@10_1_118238@10_7 | word_between=and
(6 rows)
```

Again, you can count the number of tuples in the table:

```bash
psql -d deepdive_spouse -c "select count(*) from has_spouse_features;"
```

The results should be:

      count
    ---------
     1160450

### <a name="inference_rules" href="#"></a> Writing inference rules

Now we need to tell DeepDive how to generate [factor
graphs](../../general/inference.html) to perform probabilistic inference.
We want to predict the `is_true` column of the `has_spouse` table based on the
features we have extracted, by assigning to each feature a weight that DeepDive
will learn. This is the simplest rule you can write, because it does not involve
domain knowledge or relationships among variables. 

Add the following lines to your `application.conf`, in the `inference.factors` block:

```bash
# Put your inference rules here
inference.factors {

  # A simple logistic regression rule
  f_has_spouse_features {

    # input to the inference rule is all the has_spouse candidate relations,
    #   as well as the features connected to them:
    input_query: """
      SELECT has_spouse.id      AS "has_spouse.id",
             has_spouse.is_true AS "has_spouse.is_true",
             feature
      FROM has_spouse,
           has_spouse_features
      WHERE has_spouse_features.relation_id = has_spouse.relation_id
      """

    # Factor function:
    function : "IsTrue(has_spouse.is_true)"

    # Weight of the factor is decided by the value of "feature" column in input query
    weight   : "?(feature)"
  }

  # ... (other inference rules)
}
```

This rule generates a model similar to a logistic regression classifier. We use
a set of features to make a prediction about the variable we are interested in. For
each row in the *input query* we are creating a
[factor](../../general/inference.html) that is connected to the
`has_spouse.is_true` variable and whose weight is learned by DeepDive on the
basis of the `feature` column value (i.e., of the features).

Note that the syntax requires the users to **explicitly select** in the `input_query`:

1. The `id` column for each variable
2. The variable column, which is `is_true` in this case
3. The column the weight depends on (if any), which is `feature` in this case

When selecting these column, users must explicitly alias `id` to
`[relation_name].id` and `[variable]` to `[relation_name].[variable]` in ordr
for the system to use them. For additional information, refere to the [inference
rule guide](../inference_rules.html).

### <a name="learning_inference" href="#"></a> Statistical inference and learning

Now that we have defined inference rules, DeepDive will automatically
[ground](../overview.html#grounding) the factor graph with these rules, then
perform the learning to figure out the weights, and the inference to compute
probabilities of candidate relations.

We are almost ready to run! In order to evaluate our results, we also want to
define a *holdout fraction* for our predictions. The holdout fraction defines
how much of our training data we want to treat as testing data used to compare
our predictions against. By default the holdout fraction is `0`, which means
that we cannot evaluate the precision of our results. Add the following line to
holdout one quarter of the training data:

    # Specify a holdout fraction
    calibration.holdout_fraction: 0.25

### <a name="get_result" href="#"> </a> Run and Check results

Let's try running the full pipeline using `./run.sh`. (If you want to run part
of the pipeline, refer to [pipeline section](walkthrough-extras.html#pipelines))

```bash
./run.sh
```

After running, you should see a summary report similar to:

    16:51:37 [profiler] INFO  --------------------------------------------------
    16:51:37 [profiler] INFO  Summary Report
    16:51:37 [profiler] INFO  --------------------------------------------------
    16:51:37 [profiler] INFO  ext_clear_table SUCCESS [330 ms]
    16:51:37 [profiler] INFO  ext_people SUCCESS [40792 ms]
    16:51:37 [profiler] INFO  ext_has_spouse_candidates SUCCESS [17194 ms]
    16:51:37 [profiler] INFO  ext_has_spouse_features SUCCESS [189242 ms]
    16:51:37 [profiler] INFO  inference_grounding SUCCESS [3881 ms]
    16:51:37 [profiler] INFO  inference SUCCESS [13366 ms]
    16:51:37 [profiler] INFO  calibration plot written to /YOUR/PATH/TO/deepdive/out/TIME/calibration/has_spouse.is_true.png [0 ms]
    16:51:37 [profiler] INFO  calibration SUCCESS [920 ms]
    16:51:37 [profiler] INFO  --------------------------------------------------

Great, let's take a look at some of the predictions that DeepDive computed.
DeepDive creates a view `has_spouse_is_true_inference` for each variable. Run
the following query to sample some high-confidence mention-level relations and
the sentences they come from:

```bash
psql -d deepdive_spouse -c "
  SELECT s.sentence_id, description, is_true, expectation, s.sentence
  FROM has_spouse_is_true_inference hsi, sentences s
  WHERE s.sentence_id = hsi.sentence_id and expectation > 0.9
  ORDER BY random() LIMIT 5;
"
```

The result should be something like the following (might not be the same):

```
 sentence_id |       description        | is_true | expectation | sentence
-------------+--------------------------+---------+-------------+----------
 154431@0    | Al Austin-Akshay Desai   |         |       0.972 | Guest list Jeff Atwater , Senate president Al Austin , Republican fundraiser from Tampa St. Petersburg Mayor Rick Ba
ker and wife , Joyce , St. Petersburg Brian Ballard , former chief of staff for Gov. Bob Martinez and a campaign adviser to Crist Rodney Barreto , a Crist fundraiser Ron Book , South
 Florida lobbyist Charlie Bronson , agriculture commissioner Dean Colson , a Miami lawyer and adviser to the governor on higher education issues Dr. Akshay Desai , head of Universal
Health Care Insurance in St. Petersburg Eric Eikenberg , Crist 's chief of staff Fazal Fazlin , entrepreneur -LRB- held fundraisers for Crist during the gubernatorial campaign -RRB-
, St. Petersburg .
 148797@53   | Michael-Carole Shelley   |         |       0.954 | WITH : Haydn Gwynne -LRB- Mrs. Wilkinson -RRB- , Gregory Jbara -LRB- Dad -RRB- , Carole Shelley -LRB- Grandma -RRB-
and Santino Fontana -LRB- Tony -RRB- , David Bologna and Frank Dolce -LRB- Michael -RRB- , and David Alvarez , Trent Kowalik and Kiril Kulish -LRB- Billy -RRB- .
 58364@10    | John-Jack Edwards        |         |       0.944 | This year , John and Jack Edwards are far from the only parent and child negotiating the awkward intersection of fam
ily and campaign life .
 40427@30    | James Taylor-Carly Simon |         |       0.962 | Look at James Taylor and Carly Simon , Whitney Houston and Bobby Brown or Britney Spears and Kevin Federline -LRB- y
es , calling him a musician is a stretch -RRB- .
 23278@18    | Chazz-Josh Gordon        |         |       0.944 | Directors Josh Gordon and Will Speck even discover an original way of hitting Chazz and Jimmy below the belt simulta
neously .
(5 rows)
```

We see that the results are actually pretty bad. Let's discuss how to examine
the results.

There are several common methods in DeepDive to examine results. DeepDive
generates [calibration plots](../calibration.html) for all variables defined in the
schema to help with debugging. Let's take a look at the generated calibration
plot, written to the file outputted in the summary report above
(has_spouse.is_true.png). It should look something like this:

![Calibration]({{site.baseurl}}/assets/walkthrough_has_spouse_is_true.png)

The calibration plot contains useful information that help you to improve the
quality of your predictions. For actionable advice about interpreting
calibration plots, refer to the [calibration guide](../calibration.html). 

Often, it is also useful to look at the *weights* that were learned for features
or rules. You can do this by looking at the `mapped_inference_results_weights`
table in the database. Run the following query to select the features with the highest
weight (positive features):

```bash
psql -d deepdive_spouse -c "
  SELECT description, weight
  FROM dd_inference_result_weights_mapping
  ORDER BY weight DESC
  LIMIT 5;
"
```

The results should look like the following:

                    description                 |      weight
    --------------------------------------------+------------------
     f_has_spouse_features-word_between=D-N.Y.  |  4.7886491287239
     f_has_spouse_features-word_between=married | 3.89640480091833
     f_has_spouse_features-word_between=wife    | 3.20275846390644
     f_has_spouse_features-word_between=widower | 3.18555507726798
     f_has_spouse_features-word_between=Sen.    | 2.91372149485723
    (5 rows)

Run the following query to select top negative features:

```bash
psql -d deepdive_spouse -c "
  SELECT description, weight
  FROM dd_inference_result_weights_mapping
  ORDER BY weight ASC
  LIMIT 5;
"
```

The results should be similar to:

                       description                   |      weight
    -------------------------------------------------+-------------------
     f_has_spouse_features-word_between=son          | -3.47510397136532
     f_has_spouse_features-word_between=grandson     | -3.30906093958107
     f_has_spouse_features-potential_last_name_match | -3.15563684816935
     f_has_spouse_features-word_between=Rodham       | -3.07171299387011
     f_has_spouse_features-word_between=addressing   | -2.89060819613259

Note that each execution may learn different weights, and these lists can look
different. Generally, we might see that most weights make sense while some
do not.

In the [next section](walkthrough-improve.html), we will discuss several ways to
improve the results.

