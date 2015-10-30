## The Schema By Example ##
Well, let's start with an example.

A [Flickr API](http://www.flickr.com/services/api/) method can be invoked to return JSON data. Say, [flickr.people.findByUsername](http://www.flickr.com/services/api/flickr.people.findByUsername.html) may return a JSON-formatted output on successful call:

```
{'user':{'id':'99926721@N00', 'nsid':'99926721@N00', 'username':{'_content':'mderk'}}, 
'stat':'ok'} 
```

The string above has a lot of obvious information about the structure of possible JSON reply:
  1. The structure is an object, with two attributes -- user and stat;
  1. The user attribute is an object, with three attributes -- id, nsid and username;
  1. The id and nsid are strings, and username is an object, with one attribute -- _content, which is a string;
  1. The stat attribute is a string_

I assume, any successful output of `flickr.people.findByUsername` would be of the structure described above. If we know how to parse that string (any JSON parser would do), traverse objects in the parsed result and check the types of values in them, any valid reply can act as a schema.

That's simple. If you want a schema to validate data against, simply describe the valid example. So, the schema can be defined as the very string that is specified above, or as an annotated string such as:

```
{'user':
{'id':'the user id (string)', 'nsid':'a string', 
'username':{'_content':'username content'}}, 
'stat':'status code (should be ok)'}
```

The more formal description of the schema language:
  * Strings can be defined a string literals ('anything').
  * Numeric values can be defined as number literals (`3.14`).
  * Boolean values can be defined as boolean literals (`true` or `false`).
  * Null values are `null` literals.
  * Objects are object literals (`{}`). That literal may have attributes defined (`{'foo':1}`), in which case the attributes specified are required, and no attributes other than specified are allowed.
  * Arrays are (surprise!) array literals(`[]`). The array literal can optionally define allowed types of the values in array, e.g. `['string']` or `['string', 1]`. If so, no other value types are allowed in array.

For convenience, several types can be described as strings, with '?' appended to indicate that the value can be undefined or null:
  * 'number' or 'number?' for numeric values, the latter is to indicate that the value can be skipped;
  * 'string?' for string values that can be skipped;
  * 'bool' or 'bool?' for booleans, required or not.

In my current implementation, there is no rule to indicate objects or arrays as optional, e.g. 'object?'. The problem is how we could describe an object structure in this case? The general rule is if we define an object, we should define its attributes as well. I thought about 'recursive' JSON descriptions, e.g.
`{'foo':'json:{\'type\': \'object\', \'required\': false, \'definition\':{\'bar\':1}}'}`,
but it makes life ugly, and, apart from that, who may know how deep one may dig into this recursion? The simpler solution would be to validate data against two different schemas both, one of which defined the object in question, and other -- did not. If only one of them failed, then the data is valid.

Well, to examples.
The following schemas are equivalent:
  * `{'a':'there can be a string'}, {'a': 'string'}, {'a': ''}`
  * `{'b':4563}, {'b': 0}, {'b': 'number'}`

The schemas `['string', 'number']`, `['anything', 1]`
  * is valid for `[], [3], [''], ['something', 4, 'foo']`
  * invalid for `{}, 1, '', [true], [1, false]`

The schema `{'one': 1, 'two': {'three':'string?'}}`
  * is valid for `{'one':0, 'two':{}}, {'one':0, 'two':{'three':'something'}}, {'one':2, 'two':{'three':''}}`
  * and invalid for `1, '', [], {}, {'one':0}, {'one':0, 'two':{'three':1}}, {'one':0, 'two':{}, 'foo':'bar'}` (extra attribute 'foo' in the latter case)

## What is not validated ##
Only the type of each of the data elements and the objects' structure is validated, not the actual _value_. E.g., in the Flickr example above, the value of  `stat` attribute should be 'ok', but it's not guaranteed to be so by the validator. The validator checks for the fact that the value is a string only, nothing more. The validation would fail for integer or boolean value, but would pass for 'maybe' in stat.

The problem here is similar to the 'optional objects' problem above -- the additional language should be defined, e.g. 'string;regex:^ok$', or a JSON equivalent. The simple and elegant decision doesn't come to my mind yet, so I'll leave it for tomorrow.

## Implementations ##

**JavaScript**:
[jsonvalidator.js](http://jsonvalidator.googlecode.com/svn/trunk/%20jsonvalidator/jsonvalidator.js);
Require [json.js](http://json.org/json.js)
```
var schema = '['string', 'number']'
var validator = JSONValidator(schema, true);
var isValid = validator(data) // data is a JavaScript object  or a JSON string
// isValid is data or false if invalid.
```

  * If debug parameter is passed to JSONValidator (the second argument) and is true:
```
validator = JSONValidator(schema, true);
```
> And you have Firebug installed, the error is printed to console if validation failed.

  * If throwError parameter (the second) is passed to validator function, and it's true,     then, if an error occured, it will be raised, so you can wrap the call with try .. catch   and read the error message:
```
try {
  isValid = validator(data, true);
} catch (e) {
  alert (e.message);
} 
```

**Python**:
[jsonvalidator.py](http://jsonvalidator.googlecode.com/svn/trunk/%20jsonvalidator/jsonvalidator.py);
Require [simplejson](http://cheeseshop.python.org/pypi/simplejson)

```
# JSON string
schema = '['string', 'number']'
# or an object:
schema = ['any string', 1]
validator = JSONValidator(schema);
isValid = validator.validate(data) # data is a Python object  or a JSON string
```
Raise JSONError on invalid JSON (that can not be parsed),
or JSONValidationError (no JSON parse errors, but invalid for the schema specified)