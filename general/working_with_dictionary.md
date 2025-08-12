# Working with dictionary

This doco compiles tips when working with dictionary (or similar data structure, hashmap, key-value object, etc.)

## 1. Always cater for "key not found" case.

As the title address, developer must think ahead when working with key-value data structure. I think it's easier to demonstrate this with an example in C#:

```C#
var dict = new Dictionary<string, string>();

// some operation to populate this Dictionary

// ...

// then read from the dictionary

string key = "Hello";

string value = dict[key];

// there is a chance that "Hello" doesn't exist within "dict", and accessing the value using the key would throw KeyNotFoundException.

// we should cater for this by checking if the key exists

string value;
if(dict.ContainsKey(key))
    value = dict[key];

// for brevity use ternary operator
string value = dict.ContainsKey(key) ? dict[key] : string.Empty; // give "value" a default value

// C# provides a TryGetKey method
dict.TryGetkey(key, out string value)
```

The method to deal with missing key varies from language to language, with Javascript when accessing a non existing property, it will return undefined

```javascript
var obj = {"hello": "world"};

var temp = obj.something // temp is assigned undefined

// sometime when the property is a class with properties you want to access

temp.someone; // result in TypeError

// usually, you can use optional chaining (?.)

var temp = obj.something?.someone; // assign undefined to temp
```

In short, always cater for key not found case.