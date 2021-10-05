# Dynamic Fraud Detection Engine with Apache Flink

`Dynamic Fraud Detection Engine` build on top of core [Flink](https://flink.apache.org) functionality, impls three powerful features:
- Dynamic data partitioning/shuffle at runtime, controlled based on a set of broadcast rules instead of using a hard-coded `KeySelector`.
  > Provides the ability to change how events are distributed and grouped by Flink at runtime. 
  > Such functionality often becomes a natural requirement when building jobs with dynamically reconfigurable application logic.
- Dynamic updates of fraud detection logic based on modifiable rules. 
  > Allow Flink jobs to change at runtime, without downtime from stopping and resubmitting the code.
- Low latency alerting based on custom windowing logic (without using the window API)
  > Utilize the low level [process function API](https://ci.apache.org/projects/flink/flink-docs-stable/dev/stream/operators/process_function.html) 
  > to implement low latency alerting on Windows and how to limit state growth with timers.

These features expand the possibilities of what is achievable with statically defined data flows and
provide the building blocks to fulfill complex business requirements. When used together, can eliminate
the need to recompile the code and redeploy Flink jobs for a wide range of modifications of the business logic.

## Dynamic Rule Engine

In order to fulfill complex fraud detection requirements, rules are expected to change on a frequent
basis without require recompilation of the job with each rule modification.

### Rule Schema

Rule definitions with following JSON schema. At this point, it is important to understand that `groupingKeys`
determine the actual physical grouping of events - all events with the same values of specified parameters 
(e.g. payer #25 -> beneficiary #12) have to be aggregated in the same physical instance of the evaluating operator.

```json
{
  "id": 1,
  "state": "ACTIVE",
  "groupingKeys": [
    "key1",
    "key2"
  ],
  "aggs": [
    {
      "field": "key1",
      "name": "amount",
      "func": "SUM"
    }
  ],
  "limit": "\"amount\" > 50",
  "filter": "",
  "prune": {
    "enabled": true,
    "reserved": [
      "timestamp",
      "guid"
    ]
  },
  "windowSize": 1000,
  "command": "BROADCAST_RULE"
}
```

- `state` has values of `ACTIVE`, `PAUSE`, `DELETE` and `CONTROL`, `CONTROL` will be used for a rule act as a **COMMAND**.
- `groupingKeys` used for dynamic partitioning/shuffle events.
- `aggs` arrays of aggregation with following JSON schema:
  ```json
  {
    "field": "field_name",
    "name": "amount",
    "func": "SUM"
  }
  ```
  - `field` specified which field will be aggregated.
  - `name` named this aggregation, it is a variable name maybe used in `limit` DSL expression. 
     > It'll be the field name if not be explicit specified.
  - `func` aggregation function values of `SUM`, `AVG`, `MIN`, `MAX`, `GROUP`, and maybe others in the future.
    The aggregated result will be used for evaluating the `limit` DSL expression.
  
- `limit` DSL expression for evaluating aggregations and low-latency alerting.
- `filter` DSL expression for filtering of certain events.
- `prune` configure events pruning enabled or not, and fields should be reserved when pruning enabled, pruning is default enabled.
- `windowSize` in millisecond for controlling alerting latency.
- `command` has values of `BROADCAST_RULE`, `CLEAR_STATE_ALL`, `CLEAR_STATE_ALL_STOP`, `DELETE_RULES_ALL`, and `EXPORT_RULES_CURRENT`, default value `BROADCAST_RULE`.

### Domain Specific Language (DSL)

It is easy to imagine adding extensions, such as domain specific language (DSL) for more sophisticated rule definitions, 
including filtering of certain events, logical rules chaining, and other more advanced functionality.

To investigate the core concepts of Dynamic Rule Engine and underlying DSL, let us start with articulating a realistic
sample rule definition for our fraud detection system in the form of a financial domain requirement:

> If the sum of  payments from the same payer to the same beneficiary 
> within a 4 hours period at nighttime 00:00:00 ~ 06:00:00 is greater than 200$ - trigger an alert.

Let's assume the transaction logs or events has following JSON schema:
```json
{
  "id": 8940923213314405583,
  "payeeId": 11,
  "beneficiaryId": 6,
  "payment": {
    "amount": 15.72,
    "currency": "USD"
  },
  "timestamp": 1620389265534
}
```

The financial domain requirement can be described using our rule definition:
```json
{
  "id": 1,
  "state": "ACTIVE",
  "groupingKeys": [
    "payeeId",
    "beneficiaryId"
  ],
  "aggs": [
    {
      "field": "payment.amount",
      "name": "amt",
      "func": "SUM"
    }
  ],
  "limit": "\"amt\" > 200",
  "filter": "time(\"timestamp\") >= \"00:00:00\" && time(\"timestamp\") <= \"06:00:00\"",
  "windowSize": 14400000,
  "command": "BROADCAST_RULE"
}
```

The **limit** expression `"amt" > 200`, and **filter** expression `time("timestamp") >= "00:00:00" && time("timestamp") <= "06:00:00"`
are our custom DSL expression, it is very intuitive.

#### Supported Operators & Operands

DSL supported operators and operands are:
- logical operators `&&`, `||`, `!`, or the equivalent word `and`, `or`, `not`.
- comparison operators `>`, `>=`, `<`, `<=`, `===`, `=!=`
  > NOTE: we did not use `==` and `!=` for equality comparison, we use `===`, `=!=`
- arithmetic operators `+`, `-`, `*`, `/`, `%`
- field extractor/operand `""`(double quotes), support nested. e.g. `"timestamp"`, `"amt"`, `"payment.amount"`
  > NOTE: only lhs(left hand side) `""` interpreted as field extractor
- field exist determine operand `exist("field")`  
- time interpret operands `time()`, `date()`, `datetime()`. e.g. `time("timestamp")`, `date("timestamp")`, `datetime("timestamp")`
  > the value of the field must be any one of following:
  > 1. Long timestamp value from Unix epoch
  > 2. String time value with format `HH:mm:ss` or `HH:mm:ss.SSS`
  > 3. String date value with format `yyyy-MM-dd`
  > 4. String datetime value with format `yyyy-MM-dd HH:mm:ss` or `yyyy-MM-dd'T'HH:mm:ss.SSS+0800`

#### Field Extractor/Operand & Const Operand

As a result of we use `""`(double quotes) as field extractor, we cannot write expression like `"field_name" <= "const_value"`.
To support represents string constants by double quotes, following special operands are supported:
- `field("field_name")` is an explicit field extractor
- `const(literal_value)`, e.g. `const(5)`, `const(0.5)`, `const("string_constant")`

> NOTE:
> If we want addition a number with the value of field: "field" + 5 <= 100, we SHOULD write like `field("field") + 5 <= 100`

#### Supported Time Constants

DSL expression supports time, date, datetime constants, they SHOULD be written with format:
1. time format `HH:mm:ss`
2. date format `yyyy-MM-dd`
3. datetime format `yyyy-MM-dd HH:mm:ss`

#### Special Operands for String

Special Operands for String value field, assumes some field's value are string:
- `"field" #== "prefix"` test String value field whether starts with **prefix**
- `"field" =@= "infix"` test String value field whether contains **infix**
- `"field" ==# "suffix"` test String value field whether ends with **suffix**
- `"field" =#= "regex_pattern"` test String value field whether regex matching **pattern**

#### Collection
- determine **in** simple collection 
  - `"field" in (v1,v2,v3)` test value of `field` in collection `[v1, v2, v3]`, the value can be any numerical value or string.
  - `"field" =:= (v1,v2,v3)` test value of `field` in collection `[v1, v2, v3]`, the value can be any numerical value or string
- determine **not in** simple collection
  - `"field" not in (v1,v2,v3)` test value of `field` not in collection `(v1,v2,v3)` the value can be any numerical value or string.
  - `!("field" =:= (v1,v2,v3))` test value of `field` not in collection `(v1,v2,v3)` the value can be any numerical value or string.
- high-order condition ensure in complex collection, for example:
  ```json
  {
    "a": {
        "b": {
            "key1": ["1", "2", "3"],
            "key2": [1, 2, 3, 4],
            "key3": [
                {
                    "key": "k1",
                    "val": "v1"
                },
                {
                    "key": "k2",
                    "val": "v2"
                },
                {
                    "key": "k3",
                    "val": "v3"
                },
                {
                    "key": "k4",
                    "val": "v4"
                }
            ],
            "c": {
                "k1": 10,
                "k2": 3,
                "k3": 5,
                "k5": 12
            }
        },
        "key1": 10
    },
    "key1": 100
  }
  ```
  
  - dsl supports **have** expression
    ```scala
    "a" have "key1" === 10
    "a.b.key1" have "3"
    "a.b.key2" have 3
    ```
  - dsl supports **match** expression
    ```scala
    "a.b.key3" none matches ("key" === "k5" && "val" === "v5")
    "a.b.key3" all matches (("key" #== "k") && ("val" #== "v"))
    "a.b.key3" any matches ("key" ==# "3") && ("key" =#= "^k\\d$")
    ```
  - dsl supports **match** the `unknown key name` in case like
    > The total number of payer details whoever accessed is greater than `100`, or the access COUNT of 
    > one same payer details is greater than `10` - trigger an alert.”
    
    ```scala
    ("a.b.c" value have any matches ( ? > 10)) || ("a.b.c" have size >= 100)
    ```

#### DSL preview

We provide DSL formatter and preview feature:
```scala
val expr = 1 / 2 * (x + 1)
```

will be formatted as
```
1
- * (x + 1)
2
```

## Instructions (local execution with netcat):

1. Start `netcat`:
```
nc -lk 9999
```
2. Run main method of `com.fsa.job.FlinkJob`
3. Submit `Rule` to netcat in one line JSON format

### Examples JSON:

```json
{"id":1,"state":"ACTIVE","groupingKeys":["payeeId","beneficiaryId"],"aggs":[{"field":"payment.amount","name":"amt","func":"SUM"}],"limit":"\"amt\" > 20","filter":"time(\"timestamp\") >= \"07:30:00\" && time(\"timestamp\") <= \"22:00:00\""}
```
```json
{"id":1,"state":"DELETE"}
```

### Examples of Control Commands:

```json
{"id":0,"state":"CONTROL","command":"DELETE_RULES_ALL"}
```
```json
{"id":0,"state":"CONTROL","command":"EXPORT_RULES_CURRENT"}
```
```json
{"id":0,"state":"CONTROL","command":"CLEAR_STATE_ALL"}
```

### Examples of CLI params:
--data-source kafka --rules-source kafka --alerts-sink kafka --rules-export-sink kafka

### Special functions:
- Using `COUNT` as the aggregate field for counting the events
- Using `COUNT_WITH_RESET` as the aggregate field for counting the events, and clear all events after counting.

```json
{"id":1,"state":"ACTIVE","groupingKeys":["payeeId","beneficiaryId"],"aggs":[{"field":"COUNT","name":"count","func":"SUM"}],"limit":"\"count\" > 50","filter":"time(\"timestamp\") >= \"00:00:00\" && time(\"timestamp\") <= \"06:00:00\""}
```
```json
{"id":1,"state":"ACTIVE","groupingKeys":["payeeId","beneficiaryId"],"aggs":[{"field":"COUNT_WITH_RESET","name":"count","func":"SUM"}],"limit":"\"count\" > 50","filter":"time(\"timestamp\") >= \"00:00:00\" && time(\"timestamp\") <= \"06:00:00\""}
```
