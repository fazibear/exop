# Exop [![hex.pm version](https://img.shields.io/hexpm/v/exop.svg?style=flat)](https://hex.pm/packages/exop) [![API Docs](https://img.shields.io/badge/api-docs-yellow.svg?style=flat)](http://hexdocs.pm/exop/) [![Build Status](https://img.shields.io/travis/madeinussr/exop.svg?style=flat)](https://travis-ci.org/madeinussr/exop)

Little library that provides a few macros which allow you to encapsulate business logic and validate incoming params over predefined contract.

Inspired by [Trailblazer::Operation](http://trailblazer.to/gems/operation/) - a part of awesome high-level architecture for ruby/rails applications.

## Installation

```elixir
def deps do
  [{:exop, "~> 0.3.4"}]
end
```

## Operation definition

```elixir
defmodule IntegersDivision do
  use Exop.Operation

  parameter :a, type: :integer, default: 1
  parameter :b, type: :integer, required: true,
                numericality: %{greater_than: 0}

  def process(params) do
    result = params[:a] / params[:b]
    IO.inspect "The division result is: #{result}"
  end
end
```

`Exop.Operation` provides `parameter` macro, which is responsible for the contract definition.
Its spec is `@spec parameter(atom, Keyword.t) :: none`, we define parameter name as the first atom attribute
and parameter options as the second `Keyword` attribute.

Parameter options determine a contract of a parameter, a set of parameters contracts is an operation contract.

Business logic of an operation is defined in `process/1` function, which is required by the Exop.Operation module
behaviour.

After the contract and business logic were defined, you can invoke the operation simply by calling `run/1` function:

```elixir
iex> IntegersDivision.run(a: 50, b: 5)
{:ok, "The division result is: 10"}
```

Return type will be either `{:ok, any()}` (where the second item in the tuple is `process/1` function's result) or
`{:error, {:validation, map()}}` (where the `map()` is validation errors map).

### Defined params

If for some reason you have to deal only with parameters that were defined in the contract,
you can filter out odd parameters from received Keyword/Map with `defined_params/1`

```elixir
# ...
parameter :a
parameter :b, default: 2

def process(params) do
  params |> defined_params
end
# ...

SomeOperation.run(a: 1, c: 3) # {:ok, %{a: 1, b: 2}}
```

### Interrupt

In some cases you might want to make an 'early return' from `process/1` function.
For this purpose you can call `interrupt/1` function within `process/1` and pass an interruption reason to it.
An operation will be interrupted and return `{:error, {:interrupt, your_reason}}`

```elixir
# ...
def process(_params) do
  interrupt(%{fail: "oops"})
  :ok # will not return it
end
# ...

SomeOperation.run(a: 1) # {:error, {:interrupt, %{fail: "oops"}}}
```

## Operation invocation

As said earlier, operations in most cases called by `run/1` function. This function
receives parameters collection. It's not required to pass to `run/1` function parameters
only described in the operation's contract, but only described parameters will be validated.

`run/1` function validate received parameters over the contract and if all parameters passed
the validation, the `run/1` function calls the code defined in `process/1` function.

```elixir
iex> SomeOperation.run(param1: 1, param2: "2")
_some_result_
```

If at least one of the given parameters didn't pass the validation `process/1` function's code
will not be invoked and corresponding warning in the application's log will appear.

You always can bypass the validation simply by calling `process/1` function itself, if needed.

## Parameter options

A parameter options could have various checks. Here the list of checks available yet:

* `type`
* `required`
* `default`
* `numericality`
* `in`
* `not_in`
* `format`
* `length`
* `inner`

### `type`

Checks whether a parameter's value is of declared type.

```elixir
parameter :some_param, type: :map
```

Exop handle almost all Elixir types:

* :boolean
* :integer
* :float
* :string
* :tuple
* :map
* :struct
* :list
* :atom
* :function

_Unknown type always passes this check._

### `required`

Checks the presence of a parameter in passed to `run/1` params collection.

```elixir
parameter :some_param, required: true
```

### `default`

Checks the presence of a parameter in passed to `run/1` params collection,
and if the parameter is missed - assign default value to it.

```elixir
parameter :some_param, default: "default value"
```

### `numericality`

Checks whether a parameter's value is a number and other numeric constraints.
All possible constraints are listed in the example below.

```elixir
parameter :some_param, numericality: %{equal_to: 10,
                                       greater_than: 0,
                                       greater_than_or_equal_to: 10,
                                       less_than: 20,
                                       less_than_or_equal_to: 10}
```

### `in`

Checks whether a parameter's value is within a given list.

```elixir
parameter :some_param, in: ~w(a b c)
```

### `not_in`

Checks whether a parameter's value is not within a given list.

```elixir
parameter :some_param, not_in: ~w(a b c)
```

### `format`

Checks wether parameter's value matches given regex.

```elixir
parameter :some_param, format: ~r/foo/
```

### `length`

Checks the length of a parameter's value. The value should be one of handled types:

* list (items count)
* string (chars count)
* atom (treated as string)
* map (key-value pairs count)
* tuple (items count)

`length` check is complex as `numericality` (should define map of inner checks).
All possible checks are listed in the example below.

```elixir
parameter :some_param, length: %{min: 5, max: 10, is: 7, in: 5..8}
```

### `inner`

Checks the inner of either Map or Keyword parameter. It applies checks described in `inner` map to
related inner items.

```elixir
# some_param = %{a: 3, b: "inner_b_attr"}

parameter :some_param, type: :map, inner: %{
  a: [type: :integer, required: true],
  b: [type: :string, length: %{min: 1, max: 6}]
}
```

And, of course, all checks on a parent parameter (`:some_param` in the example) are still applied.

### `struct`

Checks whether the given parameter is expected structure.

```elixir
parameter :some_param, struct: %SomeStruct{}
```

### `func`

Checks whether an item is valid over custom validation function.

```elixir
parameter :some_param, func: &__MODULE__.your_validation/1

def your_validation(param), do: !is_nil(param)
```

_it's possible to combine :func check with others (though not preferable), just make sure this check is the last check in the list_

## Validation result

If received parameters passed a contract validation, a code defined in `process/1` will be invoked.
Or you will receive `@type validation_error :: {:error, :validation_failed, map()}` as a result otherwise.
`map()` as errors reasons might look like this:

```elixir
%{param1: ["has wrong type"], param2: ["is required", "must be equal to 3"]}
```

## LICENSE

    Copyright © 2016 Andrey Chernykh ( andrei.chernykh@gmail.com )

    This work is free. You can redistribute it and/or modify it under the
    terms of the MIT License. See the LICENSE file for more details.
