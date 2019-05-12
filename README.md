# vcf-js

High performance VCF parser in pure JavaScript.

## Status

[![Build Status](https://img.shields.io/travis/com/GMOD/vcf-js/master.svg?logo=travis&style=flat-square)](https://travis-ci.com/GMOD/vcf-js)
[![Coverage Status](https://img.shields.io/codecov/c/github/GMOD/vcf-js/master.svg?style=flat-square)](https://codecov.io/gh/GMOD/vcf-js/branch/master)
[![NPM version](https://img.shields.io/npm/v/@gmod/vcf.svg?logo=npm&style=flat-square)](https://npmjs.org/package/@gmod/cram)
[![Greenkeeper badge](https://badges.greenkeeper.io/GMOD/vcf-js.svg)](https://greenkeeper.io/)

## Usage

This module is best used when combined with some easy way of retrieving the
header and individual lines from a VCF, like the `@gmod/tabix` module.

```javascript
const { TabixIndexedFile } = require('@gmod/tabix')
const VCF = require('@gmod/vcf')

const tbiIndexed = new TabixIndexedFile({ path: '/path/to/my.vcf.gz' })

async function doStuff() {
  const headerText = await tbiIndexed.getHeader()
  const tbiVCFParser = new VCF({ header: headerText })
  const variants = []
  await tbiIndexed.getLines('ctgA', 200, 300, line =>
    variants.push(tbiVCFParser.parseLine(line)),
  )
  console.log(variants)
}
```

Given a VCF with a single variant line

```text
#CHROM	POS	ID	REF	ALT	QUAL	FILTER	INFO	FORMAT	HG00096
contigA	3000	rs17883296	G	T,A	100	PASS	NS=3;DP=14;AF=0.5;DB;H2 GT:AP	0|0:0.000,0.000
```

The `variant` object returned by `parseLine()` would be

```javascript
{
  CHROM: 'contigA',
  POS: 3000,
  ID: ['rs17883296'],
  REF: 'G',
  ALT: ['T', 'A'],
  QUAL: 100,
  FILTER: 'PASS',
  INFO: {
    NS: '3',
    DP: '14',
    AF: '0.5',
    DB: null,
    H2: null,
  },
}
```

The `variant` object will also have a lazy attribute called "`SAMPLES`" that
will not be evaluated unless it is called. This can save time if you only want
the variant information and not the sample-specific information, especially if
your VCF has a lot of samples in it. In the above case the `variant.SAMPLES`
object would look like

```javascript
{
  HG00096: {
    GT: '0|0',
    AP: '0.000,0.000',
  },
},
```

The parser will try to use metadata from the header if present to convert INFO
and FORMAT values to their proper type (int, float) or split them into an
array if they represent multiple values.

Metadata can be accessed with the `getMetadata()` method. With no parameters it
will return all the data. Any parameters passed will further filter the
metadata. For example, a VCF with this header:

```text
##INFO=<ID=NS,Number=1,Type=Integer,Description="Number of Samples With Data">
##INFO=<ID=DP,Number=1,Type=Integer,Description="Total Depth">
##INFO=<ID=AF,Number=A,Type=Float,Description="Allele Frequency">
##INFO=<ID=AA,Number=1,Type=String,Description="Ancestral Allele">
##INFO=<ID=DB,Number=0,Type=Flag,Description="dbSNP membership, build 129">
##INFO=<ID=H2,Number=0,Type=Flag,Description="HapMap2 membership">
#CHROM	POS	ID	REF	ALT	QUAL	FILTER	INFO
```

You can access the VCF's header metadata like:

```javascript
> console.log(vcfParser.getMetadata())
{ INFO:
  { NS:
    { Number: 1,
      Type: 'Integer',
      Description: 'Number of Samples With Data' },
    DP: { Number: 1, Type: 'Integer', Description: 'Total Depth' },
    AF: { Number: NaN, Type: 'Float', Description: 'Allele Frequency' },
    AA: { Number: 1, Type: 'String', Description: 'Ancestral Allele' },
    DB:
    { Number: 0,
      Type: 'Flag',
      Description: 'dbSNP membership, build 129' },
    H2: { Number: 0, Type: 'Flag', Description: 'HapMap2 membership' } },}
> console.log(vcfParser.getMetadata('INFO'))
{ NS:
  { Number: 1,
    Type: 'Integer',
    Description: 'Number of Samples With Data' },
  DP: { Number: 1, Type: 'Integer', Description: 'Total Depth' },
  AF: { Number: NaN, Type: 'Float', Description: 'Allele Frequency' },
  AA: { Number: 1, Type: 'String', Description: 'Ancestral Allele' },
  DB:{ Number: 0,
    Type: 'Flag',
    Description: 'dbSNP membership, build 129' },
  H2: { Number: 0, Type: 'Flag', Description: 'HapMap2 membership' } }
> console.log(vcfParser.getMetadata('INFO', 'DP'))
{ Number: 1, Type: 'Integer', Description: 'Total Depth' }
> console.log(vcfParser.getMetadata('INFO', 'DP', 'Number'))
1
```

A list of sample names is also available in the `samples` attribute of the parser object:

```javascript
> console.log(vcfParser.samples)
[ 'HG00096' ]
```

## Breakends

If a variant line has `SVTYPE=BND`, the `ALT` field will be examined for breakend
specifications, and those will be parsed as objects.  For example:

```text
13  123456  bnd_U C C[2 : 321682[,C[17 : 198983[  6 PASS  SVTYPE=BND;MATEID=bnd V,bnd Z
```

will be parsed as

```json
{
  "CHROM": "13",
  "POS": 123456,
  "ID": [
    "bnd_U"
  ],
  "REF": "C",
  "ALT": [
    {
      "MateDirection": "right",
      "Replacement": "C",
      "MatePosition": "2 : 321682",
      "Join": "right"
    },
    {
      "MateDirection": "right",
      "Replacement": "C",
      "MatePosition": "17 : 198983",
      "Join": "right"
    }
  ],
  "QUAL": 6,
  "FILTER": "PASS",
  "INFO": {
    "SVTYPE": [
      "BND"
    ],
    "MATEID": [
      "bnd V",
      "bnd Z"
    ]
  }
}
```

## API

<!-- Generated by documentation.js. Update this documentation by updating the source code. -->

#### Table of Contents

-   [VCF](#vcf)
    -   [Parameters](#parameters)
    -   [\_parseMetadata](#_parsemetadata)
        -   [Parameters](#parameters-1)
    -   [\_parseStructuredMetaVal](#_parsestructuredmetaval)
        -   [Parameters](#parameters-2)
    -   [getMetadata](#getmetadata)
        -   [Parameters](#parameters-3)
    -   [\_parseKeyValue](#_parsekeyvalue)
        -   [Parameters](#parameters-4)
    -   [\_percentDecode](#_percentdecode)
        -   [Parameters](#parameters-5)
    -   [parseLine](#parseline)
        -   [Parameters](#parameters-6)

### VCF

Class representing a VCF parser, instantiated with the VCF header.

#### Parameters

-   `args` **[object](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Object)**
    -   `args.header` **[string](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/String)** The VCF header. Supports both LF and CRLF
        newlines.

#### \_parseMetadata

Parse a VCF metadata line (i.e. a line that starts with "##") and add its
properties to the object.

##### Parameters

-   `line` **[string](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/String)** A line from the VCF. Supports both LF and CRLF
    newlines.

#### \_parseStructuredMetaVal

Parse a VCF header structured meta string (i.e. a meta value that starts
with "&lt;ID=...")

##### Parameters

-   `metaVal` **[string](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/String)** The VCF metadata value

Returns **[Array](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Array)** Array with two entries, 1) a string of the metadata ID
and 2) an object with the other key-value pairs in the metadata

#### getMetadata

Get metadata filtered by the elements in args. For example, can pass
('INFO', 'DP') to only get info on an metadata tag that was like
"##INFO=&lt;ID=DP,...>"

##### Parameters

-   `args` **...[string](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/String)** List of metadata filter strings.

Returns **any** An object, string, or number, depending on the filtering

#### \_parseKeyValue

Sometimes VCFs have key-value strings that allow the separator within
the value if it's in quotes, like:
'ID=DB,Number=0,Type=Flag,Description="dbSNP membership, build 129"'

Parse this at a low level since we can't just split at "," (or whatever
separator). Above line would be parsed to:
{ID: 'DB', Number: '0', Type: 'Flag', Description: 'dbSNP membership, build 129'}

##### Parameters

-   `str` **[string](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/String)** Key-value pairs in a string
-   `pairSeparator` **[string](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/String)?** A string that separates sets of key-value
    pairs (optional, default `';'`)

Returns **[object](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Object)** An object containing the key-value pairs

#### \_percentDecode

Decode any of the eight percent-encoded values allowed in a string by the
VCF spec.

##### Parameters

-   `str` **[string](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/String)** A string that may contain percent-encoded characters

Returns **[string](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/String)** A string with any percent-encoded characters decoded

#### parseLine

Parse a VCF line into an object like { CHROM POS ID REF ALT QUAL FILTER
INFO } with SAMPLES optionally included if present in the VCF

##### Parameters

-   `line` **[string](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/String)** A string of a line from a VCF. Supports both LF and
    CRLF newlines.
