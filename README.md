# KBC Generic Writer

Description

**Table of contents:**  
  
[TOC]


# Functionality notes

Writes data to a specified endpoint in a specified format. Supports single table and single endpoint per configuration.

Works in two modes:

1. Basic mode - where the input data is sent to the endpoint in a specified format
2. Iteration mode - where the data is sent in iterations specified in the input data. By default 1 row = 1 iteration. 
This allows to change the endpoint dynamically based on the input using placeholders: `www.example.com/api/user/{{id}}`.
Or sending data with different user parameters that are present in the input table.

# Configuration

**Parameters**

## Path

An URL of the endpoint where the payload is being sent. 

May contain placeholders for iterations wrapped in `{{}}`,e.g. `www.example.com/api/user/{{id}}`. 
The parameter `id` needs to be specified in the `user_parameters` or in the source data itself if the column is set 
as an iteration parameter column.

## Method

Request method - POST, PUT, etc.

`"method": "POST"`

## Mode

Mode in what the data is transferred:

- JSON - input table is converted into a JSON (see json_data_config)
- BINARY - input table is sent as binary data (just like `curl --data-binary`)
- BINRAY-GZ - input is sent as gzipped binary data

**NOTE** that you need to also setup the proper request headers manually.

## json_data_config

Configuration of how the input is translated into a JSON. Supports nested structures.

The nesting is done via delimiter character, e.g.

`user_id` column will be converted to `"user":{"id": val}` object

### chunk_size

If specified, the input is being sent in chunks. When set to `1` a single object is sent `{}`, when set to >1
 an array of objects is sent `[{}, {}]`

### delimiter

A character that is used for nesting. e.g. `_`

### request_data_wrapper

A wrapper/mask of the parsed data. It needs to be json-encoded json. E.g

`"request_data_wrapper": "{ \"data\": {{data}}}"`

This will cause each request being sent as:

```json
{
  "data": [
  {"user": 
     {"id": 1}
   },
  {"user": 
     {"id": 2}
   }
  ]

}
```

### column_types

Optional configuration of column types. This version supports nesting (three levels) and three datatypes:

- `bool` -  Boolean value  case-insensitive conversion: `t`, `true`, `yes` to `True` and `f`, `false`, `no` to `False`
- `string` - String
- `number` - Number
- `object` - Object - valid JSON array or JSON object, e.g. ["1","2"], {"key":"val"}

You need to specify all columns and datatypes you want them to have in the JSON file - 
if you want the value to be enclosed in double quotes, use `string`, 
if you want the value to be numeric, use `number`. 
If you want it to be Boolean, use `bool` 
(case-insensitive conversion: `t`, `true`, `yes` to `True` and `f`, `false`, `no` to `False`)

Columns **that do not have explicitly defined datatypes** will be converted to:

- String if `infer_types_for_unknown` is set to `false` or omitted
- Datatype will be inferred from the value itself if `infer_types_for_unknown` is set to `true`

### infer_types_for_unknown

Flag whether to infer datatypes automatically from data or not.


### Example


INPUT DATA:
 
|time_reviewed_r1_r2|time_reviewed_r1_r1|time_reviewed_r2|id                                                    |field.id|ansconcat|time_submitted      |
|-------------------|-------------------|----------------|------------------------------------------------------|--------|---------|--------------------|
|True               |True               |True            |https://keboolavancouver.typeform.com/to/XXXX?id=xxxxx|123456  |Jan Palek|2019-08-13T19:05:45Z|
|True               |True               |True            |https://keboolavancouver.typeform.com/to/XXXX?id=xxxxx|123456  |Jan Palek|2019-08-13T19:05:45Z|
|True               |True               |True            |https://keboolavancouver.typeform.com/to/XXXX?id=xxxxx|123456  |Jan Palek|2019-08-13T19:05:45Z|

SETUP:

```json
"json_data_config": {
      "chunk_size": 5,
      "delimiter": "_",
      "request_data_wrapper": "{ \"data\": {{data}}}",
      "infer_types_for_unknown": true,
      "column_types": [
        {
          "column": "bool_bool2",
          "type": "number"
        },
        {
          "column": "bool_bool1",
          "type": "bool"
        },
        {
          "column": "id",
          "type": "string"
        },
        {
          "column": "field.id",
          "type": "string"
        },
        {
          "column": "ansconcat",
          "type": "string"
        },
        {
          "column": "time_submitted",
          "type": "string"
        },
        {
          "column": "id2",
          "type": "string"
        },
        {
          "column": "time_11",
          "type": "bool"
        },
        {
          "column": "time_reviewed_r2",
          "type": "bool"
        },
        {
          "column": "time_reviewed_r1_r1",
          "type": "bool"
        }
      ]
    }
```

RESULT:

```json
{
    "data": [{
            "time": {
                "submitted": "2019-08-13T19:05:45Z",
                "reviewed": {
                    "r2": true,
                    "r1": {
                        "r1": true,
                        "r2": true
                    }
                }
            },
            "ansconcat": "Jan Palek",
            "field.id": "123456",
            "id": "https://keboolavancouver.typeform.com/to/XXXX?id=xxxxx"
        }, {
            "time": {
                "submitted": "2019-08-13T19:05:45Z",
                "reviewed": {
                    "r2": true,
                    "r1": {
                        "r1": true,
                        "r2": true
                    }
                }
            },
            "ansconcat": "Jan Palek",
            "field.id": "123456",
            "id": "https://keboolavancouver.typeform.com/to/XXXX?id=xxxxx"
        }, {
            "time": {
                "submitted": "2019-08-13T19:05:45Z",
                "reviewed": {
                    "r2": true,
                    "r1": {
                        "r1": true,
                        "r2": true
                    }
                }
            },
            "ansconcat": "Jan Palek",
            "field.id": "123456",
            "id": "https://keboolavancouver.typeform.com/to/XXXX?id=xxxxx"
        }
    ]
}
```

##  user_parameters 

A list of user parameters that is are accessible from within headers and additional parameters. This is useful for storing
for example user credentials that are to be filled in a login form. Appending `#` sign before the attribute name will hash the value and store it securely
within the configuration (recommended for passwords). The value may be scalar, supported function or another scalar user parameter 
referenced by `{"attr":"par"}` object. 

Example:

```json
"user_parameters": {
      "#token": "Bearer 123456"
      "param1": 1,
      "param2" { 
         "function":"concat",
         "args":[
            "http://example.com/",
            {"attr":"param1"}
          ]
      }
}
```

You can access user parameters as:

```json
"value": {
          "attr": "#token"
        }
```

## headers
 
An array of HTTP headers to send. You may include User parameters as a value:

```json
"headers": [
      {
        "key": "Authorization",
        "value": {
          "attr": "#token"
        }
      },
      {
        "key": "Content-Type",
        "value": "application/json"
      }
    ]
```

## additional_requests_pars
 
(OPT) additional kwarg parameters that are accepted by the 
Python [Requests library](https://2.python-requests.org/en/master/user/quickstart/#make-a-request), get method. 
**NOTE** the `'false','true'` string values are converted to boolean, objects will be treated as python dictionaries

Example:

```json
"additional_requests_pars": [
      {
        "key": "verify",
        "value": "false"
      },
      {
        "key": "params",
        "value": {
          "date": {
            "attr": "date"
          }
        }
      }
    ]
```
### Request parameters

The most common usage is defining request parameters here. To send request with `?date=somedate&dryrun=true` 
use this setup:

```json
"additional_requests_pars": [
      {
        "key": "params",
        "value": {
          "date": {
            "attr": "date"
          },
          "dryrun": True
          
        }
      }
    ]
```

### Other supported parameters

- **params:** (optional) Dictionary to be sent in the query string for the `Request`.
- **cookies:** (optional) Dict object to send with the `Request`.
- **timeout:** (optional) How long to wait for the server to send
    data before giving up, as a float
- **allow_redirects:** (optional) Set to `true` by default.
- **proxies:** (optional) Dictionary mapping protocol or protocol and hostname to the URL of the proxy.
- **verify:** (optional) Either a boolean, in which case it controls whether we verify
    the server's TLS certificate, or a string, in which case it must be a path
    to a CA bundle to use. Defaults to `true`.

Note that the `date` value is taken from the `user_parameters`, which may be possibly dynamic function.


## Dynamic Functions

The application support functions that may be applied on parameters in the configuration to get dynamic values.

Currently these functions work only in the `user_parameters` scope. 
Place the required function object instead of the user parameter value.

The function values may refer to another user params using `{"attr": "custom_par"}`

**Function object**
    
```json
{ "function": "string_to_date",
                "args": [
                  "yesterday",
                  "%Y-%m-%d"
                ]
              }
```

### Function Nesting

Nesting of functions is supported:

```json
{
   "user_parameters":{
      "url":{ 
         "function":"concat",
         "args":[
            "http://example.com",
            "/test?date=",
            { "function": "string_to_date",
                "args": [
                  "yesterday",
                  "%Y-%m-%d"
                ]
              }
         ]
      }
   }
}

```

### string_to_date

Function converting string value into a datestring in specified format. The value may be either date in `YYYY-MM-DD` format,
or a relative period e.g. `5 hours ago`, `yesterday`,`3 days ago`, `4 months ago`, `2 years ago`, `today`.

The result is returned as a date string in the specified format, by default `%Y-%m-%d`

The function takes two arguments:

1. [REQ] Date string
2. [OPT] result date format. The format should be defined as in http://strftime.org/



**Example**

```json
{
   "user_parameters":{
      "yesterday_date":{
         "function":"string_to_date",
         "args":[
            "yesterday",
            "%Y-%m-%d"
         ]
      }
   }
}
```

The above value is then available in step contexts as:

```json
"to_date": {"attr": "yesterday_date"}
```

### concat

Concat an array of strings.

The function takes an array of strings to concat as an argument



**Example**

```json
{
   "user_parameters":{
      "url":{
         "function":"concat",
         "args":[
            "http://example.com",
            "/test"
         ]
      }
   }
}
```

The above value is then available in step contexts as:

```json
"url": {"attr": "url"}
```

### base64_encode

Encodes string in BASE64



**Example**

```json
{
   "user_parameters":{
      "token":{
         "function":"base64_encode",
         "args":[
            "user:pass"
         ]
      }
   }
}
```

The above value is then available in contexts as:

```json
"token": {"attr": "token"}
```


## iteration_mode

This object allows performing the requests in iterations based on provided parameters within data. The user specifies 
columns in the source table that will be used as parameters for each request. 

These will be injected in:

- Url if placeholder is specified, e.g.  `www.example.com/api/user/{{id}}`
- `user_parameters` section, any existing parameters with a same name will be replaced by the value from the data. This 
allows for example for changing request parameters dynamically `www.example.com/api/user?date=xx` where the `date` value is specified 
like:

```json
"additional_requests_pars": [
      {
        "key": "params",
        "value": {
          "date": {
            "attr": "date"
          }
        }
      }
    ]
```

### iteration_par_columns

An array of columns in the source data that will be used as parameters. Note that these columns will be excluded from the data 
payload itself.

### Example

Let's have this table on the input:

| id | date       | name  | email      | address |
|----|------------|-------|------------|---------|
| 1  | 01.01.2020 | David | d@test.com | asd     |
| 2  | 01.02.2020 | Tom   | t@test.com | asd     |

When setting iteration mode like this:

```json
"iteration_mode": {
      "iteration_par_columns": [
        "id", "date"
      ]
    }
```

**url**: `www.example.com/api/user/{{id}}`

**request params:**

```json
"additional_requests_pars": [
      {
        "key": "params",
        "value": {
          "date": {
            "attr": "date"
          }
        }
      }
    ]
```

The writer will run in two iterations:

**FIRST** With data

| name  | email      | address |
|-------|------------|---------|
| David | d@test.com | asd     |

Sent to `www.example.com/api/user/1?date=01.01.2020`




**SECOND** with data

| name  | email      | address |
|-------|------------|---------|
| Tom   | t@test.com | asd     |

Sent to `www.example.com/api/user/2?date=01.02.2020`


# Configruation example

```json
{
    "path": "https://example.com/test/{{id}}",
    "mode": "JSON",
    "method": "POST",
    "iteration_mode": {
      "iteration_par_columns": [
        "id"
      ]
    },
    "user_parameters": {
      "#token": "Bearer 123456",
    "token_encoded": {
      "function": "concat",
      "args": [
        "Basic ",
        {
          "function": "base64_encode",
          "args": [
            {
              "attr": "#token"
            }
          ]
        }
      ]
    },
      "date": {
        "function": "concat",
        "args": [
          {
            "function": "string_to_date",
            "args": [
              "yesterday",
              "%Y-%m-%d"
            ]
          },
          "T"
        ]
      }
    },
    "headers": [
      {
        "key": "Authorization",
        "value": {
          "attr": "token_encoded"
        }
      },
      {
        "key": "Content-Type",
        "value": "application/json"
      }
    ],
    "additional_requests_pars": [
      {
        "key": "params",
        "value": {
          "date": {
            "attr": "date"
          }
        }
      }
    ],
    "json_data_config": {
      "chunk_size": 1,
      "delimiter": "_",
      "request_data_wrapper": "{ \"data\": {{data}}}",
      "infer_types_for_unknown": true,
      "column_types": [
        {
          "column": "bool_bool2",
          "type": "number"
        },
        {
          "column": "bool_bool1",
          "type": "bool"
        },
        {
          "column": "id",
          "type": "string"
        },
        {
          "column": "field.id",
          "type": "string"
        },
        {
          "column": "ansconcat",
          "type": "string"
        },
        {
          "column": "time_submitted",
          "type": "string"
        },
        {
          "column": "id2",
          "type": "string"
        },
        {
          "column": "time_11",
          "type": "bool"
        },
        {
          "column": "time_reviewed_r2",
          "type": "bool"
        },
        {
          "column": "time_reviewed_r1_r1",
          "type": "bool"
        }
      ]
    },
    "debug": true
  }
```



## Development

If required, change local data folder (the `CUSTOM_FOLDER` placeholder) path to your custom path in the docker-compose file:

```yaml
    volumes:
      - ./:/code
      - ./CUSTOM_FOLDER:/data
```

Clone this repository, init the workspace and run the component with following command:

```
git clone repo_path my-new-component
cd my-new-component
docker-compose build
docker-compose run --rm dev
```

Run the test suite and lint check using this command:

```
docker-compose run --rm test
```

# Integration

For information about deployment and integration with KBC, please refer to the [deployment section of developers documentation](https://developers.keboola.com/extend/component/deployment/) 