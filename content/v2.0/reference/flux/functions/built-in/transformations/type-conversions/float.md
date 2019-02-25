---
title: float() function
description: The `float()` function converts a single value to a float.
menu:
  v2_0_ref:
    name: float
    parent: built-in-type-conversions
weight: 502
---

The `float()` function converts a single value to a float.

_**Function type:** Type conversion_  
_**Output data type:** Float_

```js
float(v: "3.14")
```

## Parameters

### v
The value to convert.

## Examples
```js
from(bucket: "sensor-data")
  |> filter(fn:(r) =>
    r._measurement == "camera" and
  )
  |> map(fn:(r) => float(v: r.aperature))
```