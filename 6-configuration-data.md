# Part 6: Configuration data



## DynamicConfigValue

* `dynamic_config_value.h`
	* Represents a tree of arbitrary config data (JSON file)
	* Uses union to represent data
	* Lots of functions to get/set keys



## Parsing JSON

* `parse_simplified_json.h`
	* Parses both regular JSON, and our "simplified" format
	* Into DynamicConfigValue

* Typical use:

```
	TempAllocator ta;
	DynamicConfigValue dcv(ta);
	const char *error = sjson::parse(buffer, dcv);
	if (error)
		return error;

	int num_cores = dcv["settings"]["cores"] || 0;
```



## Parsing with line numbers

* We want to be able to generate good error messages:

```
if (!dcv["settings"]["cores"].is_integer())
	...
```

* Somehow we need to preserve the line numbers from the original file
* parse_traced() -- `parse_simplified_json.h`
* Location is encoded and stored in `tag` bits of `dynamic_config_value.h`



## ConstConfig

* Most of the times when we compile JSON data we turn it into C structs
* But sometimes we want to preserve the "open" format
* `const_config.h`
* Packs it into a single buffer



# DynamicConfigValue -> JSON

* Used for sending messages over the console connection (for instance)
* `generate_json.h`, `generate_simplified_json.h`
* Note: SJSON is not guaranteed to survive round trip
	* We support comments in SJSON



## Config issues

* Parsing into a tree can lead to creation of a lot of objects
* This is especially problematic when using a TempAllocator
* Perhaps we should use something closer to the ConstConfig
* Not super important since it is just during compiling
