vtt.js
======

[![Build Status](https://travis-ci.org/andreasgal/vtt.js.png?branch=master)](https://travis-ci.org/andreasgal/vtt.js)

[WebVTT](https://developer.mozilla.org/en-US/docs/HTML/WebVTT) parser in JavaScript.

API
===

The parser has a simple API:

```javascript
var parser = new WebVTTParser(window, stringDecoder);
parser.onregion = function (region) {}
parser.oncue = function (cue) {}
parser.onpartialcue = function (cue) {}
parser.onflush = function () {}
parser.parse(moreData);
parser.parse(moreData);
parser.flush();
WebVTTParser.convertCueToDOMTree(window, cuetext);
WebVTTParser.processCues(window, cues);
```

The WebVTT constructor is passed a window object with which it will create new
VTTCues and VTTRegions as well as an optional StringDecoder object which
it will use to decode the data that the `parse()` function receives. For ease of
use, a StringDecoder is provided via `WebVTTParser.StringDecoder()`. If a custom
StringDecoder object is passed in it must support the API specified by the
[#whatwg string encoding](http://encoding.spec.whatwg.org/#api) spec.

`parse` hands data in some format to the parser for parsing. The passed data format
is expected to be decodable by the StringDecoder object that it has. The parser
decodes the data and reassembles partial data (streaming), even across line breaks.

```javascript
var parser = new WebVTTParser(window, WebVTTParser.StringDecoder());
parser.parse("WEBVTT\n\n");
parser.parse("00:32.500 --> 00:33.500 align:start size:50%\n");
parser.parse("<v.loud Mary>That's awesome!");
parser.flush();
```

`flush` indicates that no more data is expected and will trigger 'onflush' (see below).

`onregion` is invoked for every region that was fully parsed.

`oncue` is invoked for every cue that was fully parsed. In case of streaming parsing oncue is delayed until the cue has been completely received.

`onpartialcue` is invoked as a cue is received, and might be invoked with a cue object that only contains partial content, and might be invoked repeatedly with the same cue object in case additional streaming updates are received. After the cue was fully parsed, `oncue` will be triggered on the same cue object.

`onflush` is invoked in response to flush() and after the content was parsed completely.

`convertCueToDOMTree` parses the cue text handed to it into a tree of DOM nodes that mirrors the internal WebVTT node structure of the cue text. It uses the window object handed to it to construct new HTMLElements and returns a tree of DOM nodes attached to a top level div.

```javascript
var div = WebVTTParser.convertCueToDOMTree(window, cuetext);
```

`processCues`  converts the cuetext of the cues passed to it to DOM trees--by calling convertCueToDOMTree--and then runs the processing model steps of the WebVTT specification on the divs. The processing model applies the necessary CSS styles to the cue divs to prepare them for display on the web page.

```javascript
var divs = WebVTTParser.processCues(window, cues);
```

Browser
=======

In order to use the parser in a browser, you can build a minified version that also bundles a polyfill of
[TextDecoder](http://encoding.spec.whatwg.org/), [VTTCue](http://dev.w3.org/html5/webvtt/#vttcue-interface),
and [VTTRegion](http://dev.w3.org/html5/webvtt/#vttregion-interface) since not all browsers currently
support them. Building a browser-ready version of the library is done using `grunt` (if you haven't installed
`grunt` globally, you can run it from `./node_modules/.bin/grunt` after running `npm install`):

```
$ grunt build
Running "uglify:dist" (uglify) task
File "dist/vtt.min.js" created.

Done, without errors.
```

The file is now built in `dist/vtt.min.js` and can be used like so:

```html
<html>
<head>
  <meta charset="utf-8">
  <title>vtt.js in the browser</title>
  <script src="dist/vtt.min.js"></script>
</head>
<body>
  <script>
    var vtt = "WEBVTT\n\nID\n00:00.000 --> 00:02.000\nText",
        parser = new WebVTTParser(window, WebVTTParser.StringDecoder());
    parser.oncue = function(cue) {
      console.log(cue);
    };
    parser.onregion = function(region) {
      console.log(region);
    }
    parser.parse(vtt);
    parser.flush();
  </script>
</body>
</html>
```

Tests
=====

Tests are written and run using [Mocha](http://visionmedia.github.io/mocha/) on node.js.
Before they can be run, you need to install various dependencies:

```
$ npm install
$ git submodule update --init --recursive
```

To run all the tests, do the following:

```
$ npm test
```

If you want to run individual tests, you can install the [Mocha](http://visionmedia.github.io/mocha/) command-line
tool globally, and then run tests per-directory:

```
$ npm install -g mocha
$ cd tests/some/sub/dir
$ mocha .
```

See the [usage docs](http://visionmedia.github.io/mocha/#usage) for further usage info.

###Writing Tests###

Tests take one of two forms. Either a last-known-good JSON file is compared against a parsed .vtt file,
or custom JS assertions are run over a parsed .vtt file.

####JSON-based Tests####

JSON-based tests are useful for creating regression tests. The JSON files can be easily generated
using the parser, so you don't need to write these by hand (see details below about `cue2json`).

For example your WebVTT file could look like this:

```
WEBVTT

00:32.500 --> 00:33.500 align:start size:50%
<v.loud Mary>That's awesome!
```

The associated JSON representation might look like this:

``` json
{
  "regions": [],
  "cues": [
    {
      "id": "",
      "startTime": 32.5,
      "endTime": 33.5,
      "text": "<v.loud Mary>That's awesome!",
      "regionId": "",
      "vertical": "",
      "line": "auto",
      "snapToLines": true,
      "position": 50,
      "size": 50,
      "align": "start",
      "domTree": {
        "tagName": "div",
        "childNodes": [
          {
            "tagName": "span",
            "localName": "span",
            "title": "Mary",
            "className": "loud",
            "childNodes": [
              {
                "textContent": "That's awesome!"
              }
            ]
          }
        ],
        "style": {
          "direction": "ltr",
          "left": 50,
          "top": 0,
          "height": "auto",
          "width": 40,
          "writingMode": "horizontal-tb",
          "position": "absolute",
          "unicodeBidi": "plaintext",
          "textAlign": "start",
          "font": "5vh sans-serif",
          "color": "rgba(255,255,255,1)",
          "whiteSpace": "pre-line"
        }
      }
    }
  ]
}
```

If you use JSON you **must** define all the possible values for cue data even if they are
not being tested. Put the default values in this case. Values that exist under the "domTree"
of the parsed cue's cuetext can be left out if they are not there as the tree is generated
dynamically with no defaults for values that aren't in the cue's cuetext.

Writing the test to compare the live ouput to this JSON is done by creating a `.js` somewhere in `tests/`.
It might look like this:

```javascript
var util = require("../lib/util.js"),
    assert = util.assert;

describe("foo/bar.vtt", function(){

  it("should compare JSON to parsed result", function(){
    assert.jsonEqual("foo/bar.vtt", "foo/bar.json");
  });

});
```

The `jsonEqual` assertion does 3 kinds of checks, including:

* Parsing the specified file as UTF8 binary data without streaming (i.e., single call to `parse)
* Parsing the specified file as UTF8 binary data with streaming at every possible chunk size
* Parsing the specified file as String data without streaming (i.e., single call to `parse)

In some test situations (e.g., testing UTF8 sequences) it is impossible to parse the file as a String.
In these cases you can use `jsonEqualUTF8` instead, which does the first two checks above, but not the third.

JSON based test `.js` files can live anywhere in or below `tests/`, and the test runner will find and run them.

####Cue2json####

You can automatically generate a JSON file for a given `.vtt` file using `cue2json.js`.
You have a number of options for running `cue2json.js`.


`$ ./bin/cue2json.js tests/foo/bar.vtt` print JSON output to console.

`$ ./bin/cue2json.js tests/foo/bar.vtt > tests/foo/bar-bad.json` print JSON output to a file called `tests/foo/bar-bad.json`.

`$ ./bin/cue2json.js tests/foo/bar.vtt -j` print JSON output to a file called `tests/foo/bar.json`.

`$ ./bin/cue2json.js ./tests` walk the `tests` directory and rewrite any JSON files whose vtt source files are known.

Assuming the parser is able to correctly parse the vtt file(s), you now have the correct JSON to run
a JSON test.

**NOTE:** Since `cue2json` uses the actual parser to generate these JSON files there is the possibility that
the generated JSON will contain bugs. Therefore, always check the generated JSON files to check that the
parser actually parsed according to spec.

####JS-based Tests####

Sometimes comparing the parsed cues to JSON isn't flexible enough. In such cases, you can use JavaScript
assertions. The `lib/util.js` module provides many helper functions and objects to make this easier,
for example, being able to `parse` a `.vtt` file and get back resulting cues.

```javascript
var util = require("../lib/util.js"),
    assert = util.assert;

describe("Simple VTT Tests", function(){

  it("should run JS assertions on parsed result", function(){
    var vtt = util.parse("simple.vtt");
    assert.equal(vtt.cues.length, 1);

    var cue0 = vtt.cues[0];
    assert.equal(cue0.id, "ID");
    assert.equal(cue0.startTime, 0);
    assert.equal(cue0.endTime, 2);
    assert.equal(cue0.text, "Text");
  });

});
```

The `util.assert` object is the standard [node.js assert module](http://nodejs.org/api/assert.html) with
the addition of `jsonEqual` and `jsonEqualUTF8`. See `lib/util.js` for other testing API functions and objects.
