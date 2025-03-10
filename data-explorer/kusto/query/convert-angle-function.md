---
title: convert_angle() - Azure Data Explorer
description: This article describes convert_angle() in Azure Data Explorer.
ms.reviewer: itsagui
ms.topic: reference
ms.date: 07/03/2022
---
# convert_angle

Convert an angle value from one unit to another.

## Syntax

`convert_angle(`*value*`,`*from*`,`*to*`)`

## Arguments

| Name | Type | Required | Description |
|--|--|--|--|
| `value` | real | &check; | The value to be converted. |
| `from` | string | &check; | The unit to convert from. For possible values, see [Conversion units](#conversion-units). |
| `to` | string | &check; | The unit to convert to. For possible values, see [Conversion units](#conversion-units). |

### Conversion units

* Arcminute
* Arcsecond
* Centiradian
* Deciradian
* Degree
* Gradian
* Microdegree
* Microradian
* Millidegree
* Milliradian
* Nanodegree
* Nanoradian
* NatoMil
* Radian
* Revolution
* Tilt

## Returns

 Returns the input value converted from one angle unit to another.

## Examples

**\[**[**Click to run query**]( https://dataexplorer.azure.com/clusters/help/databases/Samples?query=H4sIAAAAAAAAAysoyswrUShKLS7NKVGwVUjOzytLLSqJT8xLz0nVMNQz0lFQd0lNL0pNVQeyHIuSczPzSktS1TUBit/6iDgAAAA=)**\]**

```kusto
print result = convert_angle(1.2, 'Degree', 'Arcminute')
```

|result|
|---|
|72|
