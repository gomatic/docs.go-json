---
title: go-json
---

**A dynamic, JSON-compatible value model for Go: a value space with typed, constant-error accessors and the coercion rules a small expression language needs. A decoded `encoding/json` document is already a `Value`, so the package interoperates directly with the standard library while supplying the typed accessors and arithmetic that raw `any` values lack.**

- **Source:** [gomatic/go-json](https://github.com/gomatic/go-json)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-json](https://pkg.go.dev/github.com/gomatic/go-json)

## Install

```
go get github.com/gomatic/go-json
```

The module path is `github.com/gomatic/go-json`; the package it provides is `value`.

```go
import value "github.com/gomatic/go-json"
```

## Value model

`Value` is a type alias for `any`, ranging over the JSON value space: `nil`, `bool`, `int64`, `float64`, `string`, `[]Value`, and `map[string]Value`. Because it is an alias — not a distinct type — a document decoded by `encoding/json` is already a `Value` with no conversion. Numbers decoded from JSON are `float64`; numeric literals may be `int64`. The accessors and arithmetic treat both uniformly.

`KindOf` reports the dynamic type as a `Kind`: `KindNull`, `KindBool`, `KindInt`, `KindFloat`, `KindString`, `KindList`, or `KindObject`. An unrecognized concrete type reports `KindNull`.

## Usage

Typed accessors extract a concrete Go type or return a sentinel error matchable with `errors.Is`:

```go
package main

import (
	"errors"
	"fmt"

	value "github.com/gomatic/go-json"
)

func main() {
	v := map[string]value.Value{
		"name": "ada",
		"age":  float64(36),
	}

	obj, err := value.AsObject(v)
	if err != nil {
		panic(err)
	}

	name, _ := value.AsString(obj["name"])
	age, _ := value.AsInt(obj["age"]) // truncates the float to int64
	fmt.Println(name, age)            // ada 36

	if _, err := value.AsString(obj["age"]); errors.Is(err, value.ErrNotString) {
		fmt.Println("age is not a string")
	}
}
```

The accessors are `AsObject`, `AsList`, `AsString`, `AsBool`, `AsInt`, and `AsFloat`. `AsInt` truncates a `float64`; `AsFloat` widens an `int64`; both numeric accessors return `ErrNotNumber` for non-numeric values.

### Coercion and arithmetic

```go
value.Truthy(nil)                 // false — nil and false are falsey; everything else is truthy
value.Truthy("")                  // true

value.Equal(int64(1), float64(1)) // true — int and float compare as numbers

n, _ := value.Compare(int64(2), float64(2.5)) // -1
s, _ := value.Compare("a", "b")               // -1 — strings compare lexically

sum, _ := value.Add(int64(2), int64(3))   // int64(5) — int stays int when both are int
mix, _ := value.Add(int64(2), float64(1)) // float64(3) — mixed operands promote to float
cat, _ := value.Add("x", int64(1))        // "x1" — a string operand concatenates, coercing the other
```

`Compare` returns `-1`, `0`, or `1`; numbers compare across int/float and strings compare lexically, while any other pairing returns `ErrIncomparable`. `Add` adds two numbers, or concatenates when either operand is a string (scalar operands are string-coerced); any other pairing returns `ErrNotNumber`.

## Errors

Every error the package emits is a constant of the sentinel type `Error`, matchable with `errors.Is`: `ErrNotObject`, `ErrNotList`, `ErrNotString`, `ErrNotNumber`, `ErrNotBool`, and `ErrIncomparable`.

## Design

The package is standard-library-only. `Value` as an alias (rather than a wrapper type) is the deliberate choice that keeps `encoding/json` interop direct — no marshal/unmarshal shims, no conversion at the boundary — while the free-function accessors supply the typed, constant-error discipline that a raw `any` cannot. It was extracted from the `value` package of `gomatic/cirql`.
