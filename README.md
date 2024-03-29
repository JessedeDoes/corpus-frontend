# BlackLab Frontend

- **[About](#About)**
    - [Intro](#Intro)
    - [Basic usage](#How-to-use)
- **[Installation](#Installation)**
    - [Requirements](#Requirements)
    - [Releases](#Download-a-release)
    - [Building from source](#Building-from-source)
    - [Using Docker](#Using-Docker)
- **[Configuration](#Configuration)**
    - [Backend configuration](#Backend-configuration)
    - [Adding corpora](#Adding-corpora)
    - [User corpora](#Allowing-users-to-add-corpora)
    - [Frontend configuration](#Frontend-configuration)
        - [Search.xml](#searchxml)
        - [Index formats](#Index-config)
        - [Custom JS](#Custom-JS)
        - [Custom CSS](#Custom-CSS)
- **[Development](#Development)**
    - [Frontend Javascript](#Frontend-Javascript)
      - [Application structure](#Application-structure)
      - [The Vuex store](#The-Vuex-store)
      - [URL generation and parsing](#URL-generation-and-parsing)
      - [Development tips](#Development-tips)
    - [Backend development](#Backend-development)



About
===================


## Intro

This is a corpus search application that works with [BlackLab Server](https://github.com/INL/BlackLab/).
At the Dutch Language Institute, we use it to publish our corpora such as [CHN](https://chn.ivdnt.org/) (CLARIN login required), [Letters as Loot](https://brievenalsbuit.ivdnt.org/) and [AutoSearch](https://portal.clarin.inl.nl/autocorp/) (CLARIN login required).


## How to use

Help is contained in the application in the form of a _page guide_ that can be opened by clicking the button on the right of the page.

![](docs/img/page_guide.png)



Installation
===================

## Requirements:

- Java 1.8
- A java servlet container such as [Apache Tomcat](https://tomcat.apache.org/).
Use Tomcat 7 version `7.0.76` or newer or Tomcat 8 version `8.0.42` or newer. Using older versions will cause some [warnings from dependencies](https://bz.apache.org/bugzilla/show_bug.cgi?id=60688).
- An instance of [BlackLab-Server](https://github.com/INL/BlackLab/).
While we do our best to make the frontend work with older versions of BlackLab, use a matching version of BlackLab (so `corpus-frontend v2.0` with `blacklab-server v2.0`).

## Download a release

Releases can be downloaded [here](https://github.com/INL/corpus-frontend/releases).

## Building from source

- Clone this repository, use `mvn package` to build the WAR file (or download the .war from the latest release) and add corpus-frontend.war to Tomcat's webapps directory.
- Optionally, create a file `corpus-frontend.properties` (name must be the same as the .war file) in the same directory as the BlackLab Server config file (e.g. `/etc/blacklab/`).
See the [Configuration section](#Backend-configuration) for more information.
- Navigate to `http://localhost:8080/corpus-frontend/` and you will see a list of available corpora you can search.

For further development and debugging help, see the [Development section](#Development).

## Using Docker

First, use docker-compose to create a blacklab-server. Then, to create this frontend run:

```
DOCKER_BUILDKIT=1 docker-compose build
docker-compose up 
```
The config file `./docker/config/corpus-frontend.properties` is mounted inside the container. See next section for the configuration details.

Configuration
====================


## Backend configuration

The backend is configured through a config file (normally `corpus-frontend.properties`). **NOTE** the path of the configfile is determined by the servlet contextPath without the leading `/`, i.e. for `/TEST/corpus-frontend` a config file with the path `TEST/corpus-frontend.properties` will be looked for in the below locations!
The config file must be located in one of the following locations (in order of priority):
- `AUTOSEARCH_CONFIG_DIR` environment variable.
- `/etc/blacklab/` (`C:\etc\blacklab` on windows)
- `/vol1/etc/blacklab/` (`C:\vol1\etc\blacklab` on windows)
- `tmp` environment variable. (usually `C:\Users\%yourusername%\AppData\Local\Temp` or `C:\Windows\Temp` on windows. `/tmp` or `/var/tmp` on linux)

Example file and defaults:

```properties

# The url under which the back-end can reach blacklab-server.
# Separate from the front-end to allow connections for proxy situations
#  where the paths or ports may differ internally and externally.
blsUrl=http://localhost:8080/blacklab-server/

# The url under which the front-end can reach blacklab-server.
blsUrlExternal=/blacklab-server/

# Optional directory where you can place files to further configure and customize
#  the interface on a per-corpus basis.
# Files should be placed in a directory with the name of your corpus, e.g. files
#  for a corpus 'MyCorpus' should be placed under 'corporaInterfaceDataDir/MyCorpus/'.
corporaInterfaceDataDir=/etc/blacklab/projectconfigs/

# For unconfigured corpora, the directory where defaults may be found (optional).
# The name of a directory directly under the corpusInterfaceDataDir.
# Files such as the help and about page will be loaded from here
#  if they are not configured/available for a corpus.
# If this directory does not exist or is not configured,
#  we'll use internal fallback files for all essential data.
corporaInterfaceDefault=default

# Path to frontend javascript files (can be configured to aid development, e.g.
#  loading from an external server so the java web server does not need
#  to constantly reload, and hot-reloading/refreshing of javascript can be used).
jspath=/corpus-frontend/js

# An optional banner message that shows above the navbar.
#  It can be hidden by the user by clicking an embedded button, and stores a cookie to keep it hidden for a week.
#  A new banner message will require the user to explicitly hide it again.
# Simply remove this property to disable the banner.
bannerMessage=<span class="fa fa-exclamation-triangle"></span> Configure this however you see fit, HTML is allowed here!

# Disable xslt and search.xml caching, useful during development.
cache=true

# Show or hide the debug info checkbox in the settings menu on the search page.
# N.B. The debug checkbox will always be visible when using webpack-dev-server during development.
# It can also be toggled by calling `debug.show()` and `debug.hide()` in the browser console.
debugInfo=false

```


## Adding corpora

Corpora may be [added manually](https://inl.github.io/BlackLab/guide/indexing-with-blacklab.html) or [uploaded by users](#Allowing-users-to-add-corpora) (if configured).

After a corpus has been added, the corpus-frontend will automatically detect it, a restart should not be required.


## Allowing users to add corpora

### Configuring BlackLab

To allow this, BlackLab needs to be configured properly (user support needs to be enabled and user directories need to be configured).
See [here](https://inl.github.io/BlackLab/server/howtos.html#let-users-manage-their-own-corpora) for the BlackLab documentation on this (scroll down a little).

When this is done, two new sections will appear on the main corpus overview page.
They allow you to define your own configurations to customize how blacklab will index your data, create private corpora (up to 10), and add data to them.

**Per corpus configuration is not supported for user corpora created through the Corpus-Frontend.**

### Formats

Out of the box, users can create corpora and upload data in any of the formats supported by BlackLab (`tei`, `folia`, `chat`, `tsv`, `plaintext` and more).
In addition, users can also define their own formats or extend the builtin formats.

### Index url

There is also a hidden/experimental page (`/corpus-frontend/upload/`) for externally linking to the corpus-frontend to automatically index a file from the web.
It can be used it to link to the frontend from external web services that output indexable files.
It requires user uploading to be enabled, and there should be a cookie/query parameter present to configure the user name.
Parameters are passed as query parameters:
```properties
file=http://my-service.tld/my-file.zip
# optional
format=folia
# optional
corpus=my-corpus-name
```

If the user does not own a corpus with this name yet, it's automatically created.
After indexing is complete, the user is redirected to the search page.


## Frontend configuration

**Per corpus configuration is not supported for user corpora.**
Though you sort of can by overriding the defaults that apply to all corpora in your instance.

By placing certain files in the `corporaInterfaceDataDir` it's possible to customize several aspects of a corpus.
Files must be placed in a subdirectory with the same name as the corpus; files for `MyCorpus` should be placed in `corporaInterfaceDataDir/MyCorpus/...`

When a file is not found for a corpus, the frontend will then check the following locations
- The directory specified in `corporaInterfaceDefault`
- [Inside the WAR](src/main/resources/interface-default/)

------------

The data directory may contain the following files and subdirectories:

- `Search.xml`
Allows you to (among other things) set the navbar links and inject custom css/js, or enable/disable pagination in documents.
See [the default configuration](src/main/resources/interface-default/search.xml).
- `help.inc`
Html content placed in the body of the `MyCorpus/help/` page.
- `about.inc`
Html content placed in the body of the `MyCorpus/about/` page.
- `.xsl` files
These are used to transform documents in your corpus into something that can be displayed on the `article/` page.
Two files can be placed here:
  - `article.xsl`, the most important one, for your document's content (previously this was `article_${formatName}.xsl` (e.g. `article_tei.xsl` or `article_folia.xsl`). This will still work for now, however, this is deprecated).
  A small note: if you'd like to enable tooltips displaying more info on the words of your corpus, you can use the `data-tooltip-preview` (when just hovering) and `data-tooltip-content` (when clicking on the tooltip) attributes on any html element to create a tooltip. Alternatively if you don't want to write your own html, you can use `title` for the preview, combined with one or more `data-${valueName}` to create a little table. `title="greeting" data-lemma="hi" data-speaker="jim"` will create a preview containing `greeting` which can be clicked on to show a table with the details`.
  - `meta.xsl` for your document's metadata (shown under the metadata tab on the page)
  **Note:** this stylesheet does not receive the raw document, but rather the results of `/blacklab-server/docs/${documentId}`, containing only the indexed metadata.
- `static/`
A sandbox where you can place whatever other files you may need, such as custom js, css, fonts, logo's etc.
These files are public, and can be accessed through `MyCorpus/static/path/to/my.file`.

---

The interface may be customized in three different ways:
- [search.xml](#searchxml)
- The config (`.blf.yaml` / `.blf.json`) used to create the corpus
- Javascript & CSS

### **Search.xml**

Allows you to set a custom display name, load custom JS/CSS, edit the shown columns for results, configure Google Analytics, and more.
See [the default configuration](src/main/resources/interface-default/search.xml) for more information.

### **Index config**

> The term format config refers to the `*.blf.yaml` or `*.blf.json` file used to index data into the corpus.

Because the format config specifies the shape of a corpus (which metadata and annotations a corpus contains, what type of data they hold, and how they are related), it made sense to let the format config specify some things about how to display them in the interface.

**NOTE:** These properties need to be set before the first data is added to a corpus, editing the format config file afterwards will not work (though if you know what your are doing, you can edit the `indexmetadata.yaml` or `indexmetadata.json` file by hand and perform the change that way).

### Through the index config you can:

- <details>
    <summary>Change the text direction of your corpus</summary>

    ```yaml
    corpusConfig:
      textDirection: "rtl"
    ```

    This will change many minor aspects of the interface, such as the order of columns in the result tables, the ordering of sentences in concordances, the text direction of dropdowns, etc.
    **NOTE:** This is a somewhat experimental feature. If you experience issues or want to see improvements, please [create an issue](https://github.com/INL/corpus-frontend/issues/new)!

    > Special thanks to [Janneke van der Zwaan](https://github.com/jvdzwaan)!

  </details>

- <details>
    <summary>Group annotations into sub sections</summary>

    ```yaml
    corpusConfig:
      annotationGroups:
        contents:
        - name: Basics
          annotations:
          - word
          - lemma
          - pos_head
        - name: More annotations
          addRemainingAnnotations: true
    ```

    The order of the annotations will be reflected in the interface.

    ![](docs/img/annotation_groups.png)

    -----------

    **NOTE:** When using annotation groups, fields not in any group will be **hidden everywhere!** (unless manually re-added through `customjs`). This includes at least the following places:
    - `Explore/N-grams`
    - `Explore/Statistics`
    - `Search/Extended`
    - `Search/Advanced`
    - `Per Hit - in the results table`
    - `Per Hit/Group by`
    - `Per Hit/Sort by`

  </details>

- <details>
    <summary>Group metadata into sub sections</summary>

    ```yaml
    corpusConfig:
      metadataFieldGroups:
      - name: Corpus/collection
        fields:
        - Corpus_title
        - Collection
        - Country
    ```

    The order of the fields & groups will be reflected in the interface.

    ![](docs/img/metadata_groups.png)

    -----------

    **NOTE:** When using metadata groups, fields not in any group will be **hidden everywhere!** (unless manually re-added through `customjs`). This includes at least the following places:
    - `Explore/Corpora`
    - `Filter`
    - `Per Hit/Group by`
    - `Per Hit/Sort by`
    - `Per Document/Group by`
    - `Per Document/Sort by`

  </details>

- <details>
    <summary>Change the display strings of values for metadata</summary>

    ```yaml
    metadata:
      fields:
      - name: year
        displayValues:
          someOtherValue: someOtherDisplayValue
          someValue: someDisplayValue
    ```

    ![](docs/img/value_display_order.png)

    Currently not supported for annotations.

  </details>

- <details>
    <summary>Designate special metadata fields</summary>

    ```yaml
    corpusConfig:
      specialFields:
        pidField: id
        titleField: Title
        authorField: AuthorNameOrPseudonym
        dateField: PublicationYear
    ```

    These fields will be used to format document title rows in the results table (unless overridden: see [customizing document titles](#customjs-configure-document-title))

    ![](docs/img/metadata_special_fields.png)

  </details>

- <details>
    <summary>Change the type of an annotation</summary>

    ```yaml
    annotatedFields:
      contents:
        annotations:
        - name: "word"
          valuePath: "folia:t"
          uiType: "text"
    ```

    ---

    ### Multiple types are supported:

    - **Text** _(default)_

      ![](docs/img/annotation_text.png)

    - **Select/Dropdown**
      Select is automatically enabled when the field does not have a uiType set, and all values are known.
      **NOTE:** Limited to `500` values! When you specify `select`, we need to know all values beforehand, BlackLab only stores the first `500` values, and ignores values longer than `256` characters. When this happens, we transform the field into a `combobox` for you, so you don't inadvertently miss any options.

      ![](docs/img/annotation_select.png)

    - **Combobox/Autocomplete**
      Just like `text`, but add a dropdown that gets autocompleted values from the server.

      ![](docs/img/annotation_combobox.png)

    - **POS** _(Part of speech)_
      **Not supported for simple search**
      This is an extension we use for corpora with split part of speech tags.
      It's mostly meant for internal use, but with some knowhow it can be configured for any corpus with detailed enough information.
      You will need to write a json file containing a `tagset` definition.
      ```typescript
      type Tagset = {
        /**
         * All known values for this annotation.
        * The raw values can be gathered from blacklab
        * but displaynames, and the valid constraints need to be manually configured.
        */
        values: {
          [key: string]: {
            value: string;
            displayName: string;
            /** All subannotations that can be used on this type of part-of-speech */
            subAnnotationIds: Array<keyof Tagset['subAnnotations']>;
          }
        };
        /**
         * All subannotations of the main annotation
        * Except the displayNames for values, we could just autofill this from blacklab.
        */
        subAnnotations: {
          [key: string]: {
            id: string;
            /** The known values for the subannotation */
            values: Array<{
              value: string;
              displayName: string;
              /** Only allow/show this specific value for the defined main annotation values (referring to Tagset['values'][key]) */
              pos?: string[];
            }>;
          };
        };
      };
      ```
      Then, during page initialization, the tagset will have to be loaded by calling

      `vuexModules.tagset.actions.load('http://localhost:8080/corpus-frontend/my-corpus/static/path/to/my/tagset.json');`
      This has to happen before $(document).ready fires! The reason for this is that the tagset you load determines how the page url is decoded, which is done when on first render.

      ![](docs/img/annotation_pos.png)
      ![](docs/img/annotation_pos_editor.png)

    - **Lexicon** _(Autocomplete for Historical Dutch)_
      Another extension we have developed for our own use. It aims to help users with searching in historical dutch texts.
      Whenever a word is entered, after a while, historical variants/spellings for that word are retrieved from our lexicon service. Those variants are then filtered against the corpus to see which actually occur. The user can then select if and with which of those variants he/she wishes to search.

      ![](docs/img/annotation_lexicon_unchecked.png)
      ![](docs/img/annotation_lexicon_checked.png)

  </details>

- <details>
    <summary>Change the type of a metadata field</summary>

    ```yaml
    metadata:
      fields:
      - name: year
        uiType: range
    ```

    - **Text** _(default)_

      ![](docs/img/annotation_text.png)

    - **Select**
      Select is automatically enabled when the field does not have a uiType set, and all values are known.
      **NOTE:** Limited to `50` values (instead of `500` - though unlike with annotations, this can be configured, see [here](https://github.com/INL/BlackLab/issues/85) for more information)!
      When you specify `select`, we need to know all values beforehand, BlackLab only stores the first `50` values, and ignores values longer than `256` characters. When this happens, we transform the field into a `combobox` for you, so you don't inadvertently miss any options.

      ![](docs/img/annotation_select.png)

    - **Combobox**
      Just like `text`, but add a dropdown that gets autocompleted values from the server.

      ![](docs/img/annotation_combobox.png)

    - **Checkbox**
      Predictably, transforms the dropdown into a checkbox selection.
      **NOTE:** The same limitations apply as with `select`.

      ![](docs/img/metadata_checkbox.png)

    - **Radio**
      Like checkbox, but allow only one value.
      **NOTE:** The same limitations apply as with `select`.

      ![](docs/img/metadata_radio.png)

    - **Range**
      Use two inputs to specify a range of values (usually for numeric fields, but works for text too).

      ![](docs/img/metadata_range.png)

    - **Multi-field Range** _(custom js only!)_
      This is a special purpose field that can be used if your documents contain metadata describing a value in a range.

      ![](docs/img/metadata_multi_range.png)

      For example: your documents have an unknown date of writing, but the date of writing definitely lies in some known range, for example between 1900-1950. This data is stored in two fields; `date_lower` and `date_upper`.
      ```javascript
        vuexModules.filters.actions.registerFilter({
        filter: {
          componentName: 'filter-range-multiple-fields',
          description: 'Filters documents based on their date range',
          displayName: 'Date text witness',
          groupId: 'Date', // The filter tab under which this should be placed, missing tabs will be created
          id: 'my-date-range-filter', // a unique id for internal bookkeeping
          metadata: { // Info the widget needs to do its work
            low: 'date_lower', // the id of the metadata field containing the lower bound
            high: 'date_upper', // the id of the metadata field containing the upper bound
            mode: null, // allowed values: 'strict' and 'permissive'. When this is set, hides the 'strictness' selector, and forces strictness to the set mode
          }
        },
        // Set the id of another filter here to append this filter behind that one.
        // Undefined will add the filter at the top.
        precedingFilterId: undefined
      });
      ```

      The `Permissive`/`Strict` mode (see image) toggles whether to match documents that merely overlap with the provided range, or only documents that fall fully within the range.
      E.G.: the document has `date_lower=1900` and `date_upper=1950`. The query is `1900-1910`, this matches when using Permissive (as the values overlap somewhat), while Strict would not match, as the document's actual value could also be outside this range. To also match using Strict, the query would have to be at least `1899-1951`.

    - **Date**
      A calendar-based filter for dates. 
      
    
      ![](docs/img/metadata_date.png)

      It expects dates to be formatted as YYYYMMDD e.g. 20220403 for 3rd of april 2022.  
      It expects a leading zero for dates before the year 1000.
      It works much like the `Multi-field Range` filter, only for full dates instead of only a single number.

      Config as follows:
        ```javascript
        vuexModules.filters.actions.registerFilter({
            filter: {
              componentName: 'filter-date',
              // description: 'Filters documents based on their date range',
              // displayName: 'Date text witness',
              // groupId: 'Date', // The filter tab under which this should be placed, missing tabs will be created
              id: 'my-date-range-filter', // a unique id for internal bookkeeping
              metadata: { // Info the widget needs to do its work
                low: 'date_lower', // the id of the metadata field containing the lower bound
                high: 'date_upper', // the id of the metadata field containing the upper bound
                mode: null, // allowed values: 'strict' and 'permissive'. When this is set, hides the 'strictness' selector, and forces strictness to the set mode
              }
            },
        ```

        > Tip: Use blacklab's `concatDate` process  
          `concatDate`: concatenate 3 separate date fields into one, substituting unknown months and days with the first or last possible value. The output format is YYYYMMDD. Numbers are padded with leading zeroes.  
          Requires 4 arguments:  
            * `yearField`: the metadata field containing the numeric year  
            * `monthField`: the metadata field containing the numeric month (so "12" instead of "december" or "dec")  
            * `dayField`: the metadata field containing the numeric day  
            * `autofill`: `start` to autofill missing month and day to the first possible value (01), or `end` to autofill the last possible value (12 for months, last day of the month in that year for days - takes in to account leap years).
            This step requires that at least the year is known. If the year is not known, no output is generated.  
  </details>

### **Custom JS**

A custom javascript file can be injected by setting it in [search.xml](#searchxml)
> **NOTE:** your javascript file is shared between all pages!
This means the vuex store might not be available! Check which page you're on beforehand by using the url or detecting whether the store exists and what exists inside it.
All javascript should run _before_ `$(document).ready` unless otherwise stated.


Through javascript you can do many things, but outlined below are some of the more interesting/useful features on the `/search/` page:

- <details>
    <summary>[Global] - Hide the page guide</summary>

    `vuexModules.ui.actions.global.pageGuide.enable(false)`
  </details>

- <details>
    <summary>[Global] - Configure which annotations & metadata fields are visible</summary>

   >  This is just a shorthand method of configuring several parts of the UI. Individual features can also be configured one by one. Refer to the [source code](src/frontend/src/store/search/ui.ts) for more details (all of this module's exports are exposed under `window.vuexModules.ui`).

    First run (from the browser console) `printCustomJs()`.
    You will see (approximately) the following output. The printed javascript  reflects the current settings of the page.

    ```js
    var x = true;
    var ui = vuexModules.ui.actions;
    ui.helpers.configureAnnotations([
      [                   ,    'EXTENDED'    ,    'ADVANCED'    ,    'EXPLORE'    ,    'SORT'    ,    'GROUP'    ,    'RESULTS'    ,    'CONCORDANCE'    ],

      // Basics
      ['word'             ,        x         ,        x         ,        x        ,      x       ,       x       ,                 ,                     ],
      ['lemma'            ,        x         ,        x         ,        x        ,              ,               ,                 ,                     ],
      ['pos'              ,        x         ,        x         ,        x        ,              ,               ,                 ,                     ],
      // (not in any group)
      ['punct'            ,                  ,                  ,                 ,              ,               ,                 ,                     ],
      ['starttag'         ,                  ,                  ,                 ,              ,               ,                 ,                     ],
      ['word_id'          ,                  ,                  ,                 ,              ,               ,                 ,          x          ],
      // ...
    ]);

    ui.helpers.configureMetadata([
      [                                      ,    'FILTER'    ,    'SORT'    ,    'GROUP'    ,    'RESULTS/HITS'    ,    'RESULTS/DOCS'    ,    'EXPORT'    ],

      // Date
      ['datering'                            ,                ,      x       ,       x       ,                      ,          x           ,                ],
      ['decade'                              ,                ,      x       ,       x       ,                      ,                      ,                ],

      // Localization
      ['country'                             ,       x        ,      x       ,       x       ,                      ,                      ,                ],
      ['region'                              ,       x        ,      x       ,       x       ,                      ,                      ,                ],
      ['place'                               ,       x        ,      x       ,       x       ,                      ,                      ,                ],
      // ...
    ]);
    ```

    You can now paste this code into your corpus's `custom.js` file and edit the cells.

    The `configureAnnotations` columns configure the following:
    - **EXTENDED**: Make the annotation appear in the `Extended` search tab.
    - **ADVANCED**: Make the annotation appear in the `Advanced/QueryBuilder` search tab.
    - **EXPLORE**: Make the annotation one of the options in the appear in the the `N-Gram` form.
    - **SORT**: Make this annotation one of the options in the `Sort by` dropdown (below the `hits` results table). _(Requires the forward index to be enabled for this annotation)_
    - **GROUP**: Make this annotation available for grouping. This affects whether it is shown in the `Group by` dropdown options above the results table, and in the `N-Gram` and `Statistics` dropdowns in the `Explore` forms. _(Requires the forward index to be enabled for this annotation)_
    - **RESULTS**: Make a separate column in the `hits` table that shows this annotation's value for every hit (typically enabled by default for `lemma` and `pos`).
    - **CONCORDANCE**: Show the value for this annotation when opening a hit's details in the `hits` table.

    Similarly, these are the meanings of the columns in `configureMetadata`:
    - **FILTER**: Make this metadata field available as a filter. (Filters are shown for `Extended`, `Advanced/QueryBuilder`, `Expert`, and all `Explore` forms.
    - **SORT**: Make this metadata field one of the options in the `Sort by` dropdown (below the `Per hit` and `Per document` results tables.
    - **GROUP**: Make this metadata field avaiable for grouping. This affects whether it is shown in the `Per hit`, `Per document`, and `Statistics` form under `Explore`.
    - **RESULTS/HITS**: Show a column in the `Per hit` table detailing the metadata for every hit's document.
    - **RESULTS/DOCS**: Show a column in the `Per document` table with the document's metadata.
    - **EXPORT**: Whether to include this metadata in result exports.

    Any columns that are completely empty (not one annotation/metadata is checked) are left at their default values. For most things this means everything in an annotation/metadata group is shown, unless no groups are defined, in which case everything not marked `isInternal` is shown.

    If you want to hide every single option in a category, use the dedicated function to configure that section of the ui and pass it an empty array.

  </details>

- <details>
    <summary>Group metadata into sections with more control</summary>
    <a name="customjs-configure-metadata-subheadings"></a>
    With customJS you can do two things you can't do with just config: 
    - automatically activate filters when a filter tab is open
    - add subheadings in between filters.

    In this way you can divide your corpus into `sections` if you will, by giving 
    every section its own tab in the `filters` section, and setting an automaticaly activating filter 
    for that section.

    ![](docs/img/ metadata_groups_with_sections.png)

    ```typescript
    type FilterGroup = {
      tabname: string;
      subtabs: Array<{
        tabname?: string;
        fields: string[];
      }>;
      query?: MapOf<string[]>;
    }
    ```


    ```javascript
    const filterState = vuexModules.filters.getState();
    filterState.filterGroups = [{
      tabname: 'My Filters' 
      subtabs: [{
        tabname: 'Title', // filters pertaining to titles
        fields: ['newspaper_title', 'section_title', 'article_title'] // names of metadata fields in your source material
      }, {
        tabname: 'Author', // filters pertaining to author...
        fields: ['author_name', 'author_birthdate']
      }],
      query: {
        some_metadata_field: ['Some', 'Values'], // force the filter some_metadata_field to the value ['Some', 'Values'] while the tab 'My Filters' is opened
        // ... other fields here
      }
    }, {
      // Second tab .... follows the same config. The same filter may occur multiple times in different tabs.
    }]
    ```

    -----------

    **NOTE:** When using metadata groups, fields not in any group will be **hidden everywhere!** (unless manually re-added through `customjs`). This includes at least the following places:
    - `Explore/Corpora`
    - `Filter`
    - `Per Hit/Group by`
    - `Per Hit/Sort by`
    - `Per Document/Group by`
    - `Per Document/Sort by`

  </details>

- <details>
    <summary>[Global] - Set custom error messages</summary>
    <a name="customjs-configure-error-message"></a>

    A function that is called whenever an error is returned from BlackLab. 
    Context is one of 'snippet', 'concordances', 'hits', 'docs', 'groups' detailing during what action the error occured.

    `vuexModules.ui.getState().global.errorMessage = function(error, context) { return error.message; }`
    Error looks like
    ```ts
    type ApiError = {
      /** Short summary */
      title: string;
      /** Full error */
      message: string;
      /** Message for HTTP status, e.g. "Service unavailable" for 503 */
      statusText: string;
    }
    ```
  </details>


- <details>
    <summary>[Search] - Hide the query builder</summary>

    `vuexModules.ui.actions.search.advanced.enable(false)`
  </details>

- <details>
    <summary>[Search] - Show/hide Split-Batch and Within</summary>

    `vuexModules.ui.actions.search.extended.splitBatch.enable(false)`
    `vuexModules.ui.actions.search.extended.within.enable(false)`

    It's also possible to set which tags are shown (and how) in `within`.
    You can only add tags that you actually index (using the [inlineTags options](https://inl.github.io/BlackLab/guide/how-to-configure-indexing.html#annotated-configuration-file) in your index config yaml)
    ```js
    vuexModules.ui.actions.search.extended.within.elements({
      title: 'Tooltip here (optional)',
      label: 'Sentence',
      value: 's'
    });
    ```
  </details>

- <details>
    <summary>[Search] - Show/hide Split-Batch and Within</summary>

    `vuexModules.ui.actions.search.extended.splitBatch.enable(false)`
    `vuexModules.ui.actions.search.extended.within.enable(false)`

    It's also possible to set which tags are shown (and how) in `within`.
    You can only add tags that you actually index (using the [inlineTags options](https://inl.github.io/BlackLab/guide/how-to-configure-indexing.html#annotated-configuration-file) in your index config yaml)
    ```js
    vuexModules.ui.actions.search.extended.within.elements({
      title: 'Tooltip here (optional)',
      label: 'Sentence',
      value: 's'
    });
    ```
  </details>

- <details>
    <summary>[Results] - Show/hide the export buttons</summary>
    <a name="customjs-toggle-export"></a>

    `vuexModules.ui.actions.results.exportEnabled(false)`
  </details>

- <details>
    <summary>[Results/Hits] Enable custom inserts in concordances, such as an audio player for spoken corpora</summary>

    The following example will create small play buttons in concordances, allowing the user to listen to the fragment. We use this feature in the `CGN (Corpus Gesproken Nederlands)` corpus.

    ![](docs/img/concordance_audio_player.png)

    To enable this, three things are needed:
    - The corpus needs to be annotated with some form of link to an audio file in the document metadata (or token annotations).
    - You need to have hosted the audio files somewhere.
      **NOTE:** The `/static/` directory will not work, as the server needs to support the http `range` headers, which that implementation does not. (PR would be most welcome!)
    - A function that transforms a result into a start- and endtime inside the audio file.

    Shown here is (an edited version of) the function we use in the `CGN`, but specific implementation will of course be unique to your situation.
    ```javascript
    
    /*  The context object contains the following information:
      {
        corpus: string, 
        docId: string, // the document id
        snippet: BLTypes.BLHitSnippet, // the raw hit info as returned by blacklab
        document: BLTypes.BLDocInfo, // the document metadata
        documentUrl: string, // url to view the document in the corpus-frontend
        wordAnnotationId: string, // configured annotation to display for words (aka vuexModules.ui.results.hits.wordAnnotationId)
        dir: 'ltr'|'rtl',
        citation: {
          left: string;
          hit: string;
          right: string;
        }
      }

      The returned object should have the following shape:
      {
        name: string; // unique name for the widget you're rendering, can be anything
        component?: string; // (optional) name of the Vue component to render, component MUST be globally installed using vue.component(...)
        element?: string; // when not using a vue component, the name of the html element to render, defaults to 'div'
        props?: any; // attributes on the html element (such as 'class', 'tabindex', 'style' etc.), or props on the vue component
        content?: string // html content of the element, or content of the default slot when using a vue component
        listeners?: any; // event listeners, passed to v-on, so 'click', 'hover', etc. 
      }
    */
    vuexModules.ui.getState().results.hits.addons.push(function(context) {
      var snippet = context.snippet;
      var docId = context.docId;
      var s = 'begintime'; // word property that stores the start time of the word
      var e = 'endtime'; // word property that stores the end time of the word
      
      // find the first word that has a start time defined, and the first word that has an end time defined
      var startString = snippet.left[s].concat(snippet.match[s]).concat(snippet.right[s]).find(function(v) { return !!v; });
      var endString = snippet.left[e].concat(snippet.match[e]).concat(snippet.right[e]).reverse().find(function(v) { return !!v; });

      // Returning undefined will disable the addon for this hit
      if (!startString && !endString) {
        return undefined;
      }

      // Transform format of HH:MM:SS.mm to a float
      var start = startString ? startString.split(':') : undefined;
      start = start ? Number.parseInt(start[0], 10) * 3600 + Number.parseInt(start[1], 10) * 60 + Number.parseFloat(start[2]) : 0;
      
      // Transform format of HH:MM:SS:mm to a float
      var end = endString ? endString.split(':') : undefined;
      end = end ? Number.parseInt(end[0], 10) * 3600 + Number.parseInt(end[1], 10) * 60 + Number.parseFloat(end[2]) : Number.MAX_VALUE;

      return {
        component: 'AudioPlayer', // name for our component
        name: 'audio-player', // render the vue component 'audio player', so <audio-player>
        props: { // with these props, so <audio-player :docId="..." :startTime="..." :endTime="..." :url="...">
          docId: docId,
          startTime: start,
          endTime: end,
          url: config.audioUrlBase + docId + '.mp3'
        },
      }
   })
    ```
  </details>

- <details>
    <summary>[Results/Hits] configure which annotation is shown as context and snippet and enable html mode</summary>

    `vuexModules.ui.actions.results.shared.concordanceAnnotationId('word_xml')`
    `vuexModules.ui.actions.results.shared.concordanceAsHtml(true)`

    `concordanceAnnotationId` lets you set which annotation is used to display words in the results table:

    1.  ![](docs/img/concordance_annot_word.png)
    2.  `vuexModules.ui.actions.results.shared.concordanceAnnotationId('pos_id')`
    3.  ![](docs/img/concordance_annot_posId.png)

    For a more interesting example consider the following:

    In diacritic words, there are usually two words, but only one lemma shared between them.
    Indexing multiple words per token is supported by blacklab,
    but due to a limitation in the forward index, only the first value for an annotation can be shown in the results.
    Using this system you can use a different annotation for searching (with multiple indexed values)
    than you use for displaying the results (with the multiple words concatenated for example).

    -------

    `concordanceAsHtml` is an advanced feature.
    Combined with blacklab's [captureXml](https://github.com/INL/BlackLab/blob/9fdc0e146f136287b0c3cca8456b0ef60ce2cbe2/core/src/site/markdown/how-to-configure-indexing.md#indexing-xml) mode, you can index snippets of raw xml from your document into an annotation. You can then set that annotation to display as html in the results table. This allows you to style it as any html using `custom css`.

    Internally, we use this to display strike throughs, manual corrections and underlines in words in historical corpora.
    For example: given a word structure like so:
    ```xml
      <!-- source document -->
      <w>
        en<strikethrough>de</strikethrough>
      </w>
    ```

    ```yaml
      # Index config
      - name: word_xml
        valuePath: //w
        captureXml: true
    ```

    ```typescript
      // Custom js
      vuexModules.ui.actions.results.shared.concordanceAnnotationId('word_xml')
      vuexModules.ui.actions.results.shared.concordanceAsHtml(true)
    ```

    ```css
      /* Custom css */
      strikethrough {
        text-decoration: line-through;
      }
    ```
    > en<del>de</del>

    Browser support for non-html elements is good, so arbitrary xml should work out of the box, given it is well-formed.

    USE THIS FEATURE WITH CARE! It may break your page if the xml contains unexpected or malformed contents.

  </details>

- <details>
    <summary>[Results/Docs] Customize the display of document titles in the results table</summary>
    <a name="customjs-configure-document-title"></a>
    
    By setting a callback to generate or extract the title of the documents, you can have more control over it.
    

    ```js
    /**
     * @param metadata all metadata of the document, in the form of { [fieldName: string]: string[] }
     * @param specialFields the names of the pid, title, author, and date fields, in the shape of 
      { 
        authorField: string, 
        pidField: string, 
        dateField: string, 
        titleField: string 
      }
      @returns {string}
     */
    vuexModules.ui.getState().results.shared.getDocumentSummary = function(metadata, specialFields) {
      return 'The document title is: ' + metadata[specialFields.titleField][0];
    }
    ```
    ![](docs/img/metadata_special_fields.png)

  </details>

- <details>
    <summary>Miscellaneous configuration</summary>
    
    ## Hide the small suffixes in the groupBy dropdown
    This is currently not supported in other dropdowns.

    `vuexModules.ui.actions.dropdowns.groupBy.annotationGroupLabelsVisible(false)`
    `vuexModules.ui.actions.dropdowns.groupBy.metadataGroupLabelsVisible(false)`
    
    | With labels | Without labels |
    |:---:|:---:|
    | ![](docs/img/groupby_with_labels.png) | ![](docs/img/groupby_without_labels.png) |

    ## Set the interval and duration of the totals counter
    Set for how long the spinner will keep automatically refreshing until the user must continue it.  
    `<=0` will keep refreshing indefinitely until the query finishes.  
    `vuexModules.ui.results.shared.totalsTimeoutDurationMs(90_000)`  
    Set how quickly the spinner will refresh while it's polling. This setting has a minimum of 100ms.  
    `vuexModules.ui.results.shared.totalsRefreshIntervalMs(2_000)`  
    
    ![](docs/img/result_totals_in_progress.png)
    

  </details>

- <details>
    <summary>Override any other data you want</summary>

    If you know what you're doing, it's possible to override any value in the store.
    The `corpus` module (`vuexModules.corpus`) contains all data used to render the page, including the annotations, metadata fields, display names, descriptions, uiTypes, etc.
    This object is writable and can be interacted with from the console or javascript, so tinker away.

    Changing the ids of annotations/metadata will result in broken queries though! :)

  </details>

----

The `/article/` page has other features that can be enabled.
Enabling any of these will show a new `Statistics` tab next to the default `Content` and `Metadata` tabs.

- <details>
    <summary>A table with whatever data you wish to show.</summary>

    You may provide a function with the signature `(BLDocument, BLHitSnippet) => { [key: string]: string; }` (see [blacklabTypes](../src/types/blacklabtypes.ts) for type definitions)
    ```javascript
    vuexModules.root.actions.statisticsTableFn(function(document, snippet) {
        var ret = {};
        ret['Tokens'] = document.docInfo.lengthInTokens;
        ret['Types'] = Object.keys(snippet.match['pos'].reduce(function(acc, v) {
            acc[v] = true;
            return acc;
        }, {})).length;
        ret['Lemmas'] = Object.keys(snippet.match['lemma'].reduce(function(acc, v) {
            acc[v] = true;
            return acc;
        }, {})).length

        var ratio = ret['Tokens'] / ret['Types'];
        var invRatio = 1/ratio;
        ret['Type/token ratio'] = '1/'+ratio.toFixed(1)+' ('+invRatio.toFixed(2)+')';

        return ret;
    });
    ```
    ![](docs/img/article_table.png)

  </details>

- <details>
    <summary>A pie chart displaying the frequency of an annotation's values</summary>

    ```javascript
    vuexModules.root.actions.distributionAnnotation({
        displayName: 'Token/Part of Speech Distribution',
        id: 'pos_head'
    });
    ```
    ![](docs/img/article_pie.png)

  </details>

- <details>
    <summary>A graph showing growth of annotations in the document</summary>

    ```javascript
    vuexModules.root.actions.growthAnnotations({
        displayName: 'Vocabulary Growth',
        annotations: [{
            displayName: 'Word types',
            id: 'word'
        }, {
            displayName: 'Lemmas',
            id: 'lemma'
        }],
    });
    ```
    ![](docs/img/article_chart.png)

  </details>

- <details>
    <summary>The color palette for the charts</summary>

    If you're using custom css, this can help them blend in with your own style.
    <span style="color: white; background: #337ab7; display: inline-block; border-radius: 3px; padding: 0 0.5em;">Defaults to bootstrap 3 primary blue (`#337ab7`).</span>

    ```javascript
    vuexModules.root.actions.baseColor('#9c1b2e');
    ```

  </details>

### **Custom CSS**

We have included a template [SASS](https://sass-lang.com/) file [here](src/frontend/src/style-template.scss) to allow you to customize your page's color theme easily.
From there you can then add your own customizations on top.

Create a file with the following contents

```scss
// custom.scss

$base: hsl(351, 70%, 36%); // Defines the base color of the theme, this can be any css color
@import 'style-template.scss'; // the absolute or relative path to our template file

// Your own styles & overrides here ...

```
You now need to compile this file by following the following steps:
- Install [Node.js](https://nodejs.org/en/)
- Install the `node-sass` package by running `npm install -g node-sass`
- Compile the file by running `node-sass -o . custom.scss`

You will now have a `custom.css` file you can include in your install through `search.xml`.

Development
===================

## Frontend Javascript

The app is primarly written in [Vue.js](https://vuejs.org/).
Outlined here is the `/search/` page, as it contains the majority of the code.

### **Application structure**

Entry points are the following files
- [article.ts](src/frontend/src/article.ts)
  Handles the hit navigation, graphs and charts on the document page `/corpus-frontend/docs/...`
- [corpora.ts](src/frontend/src/corpora.ts)
  The main index page (or `/corpus-frontend/corpora/`)
- [remote-index.ts](src/frontend/src/remote-index.ts)
  The `/upload/` page.
- [search.ts](src/frontend/src/search.ts)
  The search form

Individual components are contained in the [pages](src/frontend/src/pages) directory. These components are single-use and/or connected to the store in some way.
The [components](src/frontend/src/components) directory contains a few "dumb" components that can be reused anywhere.

### **The Vuex store**

We use [vuex](https://vuex.vuejs.org/guide/) to store the app state, treat it as a central database (though it's not persisted between sessions).
The vuex store is made up of many `modules` that all handle a specific part of the state, such as the metadata filters, or the settings menu (page size, random seed).

The [form](src/frontend/src/store/search/form) directory contains most of the state to do with the top of the page, such as filters, query builder, explore view.
The [results](src/frontend/src/store/search/results) directory handles the settings that directly update the results, such as which page is open, how results are grouped, etc.

A couple of modules have slightly different roles:
- The [corpus](src/frontend/src/store/search/corpus.ts) module stores the blacklab index config and is used almost everywhere.
- The [history](src/frontend/src/store/search/history.ts) module stores the query history (_not the browser history_).
- The [query](src/frontend/src/store/search/query.ts) module contains a snapshot of the form (with filters, patterns, etc) as it was when `submit` was pressed.
  This is what actually determines the results being shown, and is what render the query summary etc.
- The [tagset](src/frontend/src/store/search/tagset.ts) module is mostly inactive, but it stores the info to build the editor for the `pos` uiType.

### **URL generation and parsing**

The current page url is generated and updated in [streams.ts](src/frontend/src/store/search/streams.ts).
It contains a few things: a stream that listens to state changes in the `vuex` store, and is responsible for updating the page url, and a couple streams that fetch some metadata about the currently selected/searched corpus (shown below the filters and at the top of the results panel).
![](docs/img/filter_overview.png)
![](docs/img/result_totals.png)

Url parsing happens in the [UrlStateParser](src/frontend/src/store/search/util/url-state-parser.ts).
The url parsing is a little involved, because depending on whether a `tagset` is provided it can differ (the cql pattern is normally parsed and split up so we know what to place in the `simple` and `extended` views, but this needs to happen differently when a tagset is involved).
Because of this, the store is first initialized (with empty values everywhere), then the url is parsed, after which the state is updated with the parsed values (see [search.ts](src/frontend/src/search.ts)).
When navigating back and forth through browser history, the url is not parsed, instead the state is attached to the history entry and read directly.

### **Development tips**

Install the Vue devtools! ([chrome](https://chrome.google.com/webstore/detail/vuejs-devtools/nhdogjmejiglipccpnnnanhbledajbpd), [firefox](https://addons.mozilla.org/en-US/firefox/addon/vue-js-devtools/)).

You can compile and watch the javascript files using webpack.
Execute `npm run start` in the `src/frontend/` directory. This will start `webpack-dev-server` (webpack is a javascript build tool) that will serve the compiled files (the [entry points](#structure)) at `localhost/dist/`.
It adds a feature where if one of those files is loaded on the page, and the file changes, your page will reload automatically with the new changes.

Combining this with `jspath` in `corpus-frontend.properties` we can start the corpus-frontend as we normally would, but sideload our javascript from `webpack-dev-server` and get realtime compilation :)

```properties
# No trailing slash!
jspath=http://localhost:80/dist
```
```bash
cd corpus-frontend/src/frontend/
npm run start
```

One note is that by default the port is `8080`, but we changed it to `80`, as `tomcat` already binds to `8080`. To change this, edit the `scripts.start` property in [package.json](src/frontend/package.json).


## Backend development

The backend is written in Java, and does comparitively little.
Its most important tasks are serving the right javascript file and setting up a page skeleton (with [Apache Velocity](https://velocity.apache.org/)).

When a request comes in, the `MainServlet` fetches the relevant corpus data from BlackLab, reads the matching `search.xml` file, and determines which page to serve (the `*Response` classes). Together this renders the header, footer, defines some client side variables (mainly urls to the corpus frontend server and blacklab servers).
From there on out the rest happens clientside.

It also handles most of the `document` page, retrieving the xml and metadata and converting it to html.

<br>
<br>
<br>

-----------

If you have any further questions or experience any issues, please contact [Jan Niestadt](mailto:jan.niestadt@ivdnt.org) and/or [Koen Mertens](mailto:koen.mertens@ivdnt.org).

Like BlackLab, this corpus frontend is licensed under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).
