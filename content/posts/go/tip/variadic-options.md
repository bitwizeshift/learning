+++
title = 'Variadic Options'
date = 2026-07-25
id = 'GO-TIP-001'
slug = 'GO-TIP-001'
aliases = ['/tip/go-tip-001', '/go-tip-001']
resources = ['tip']
toolchains = ['go']
concepts = ['design']

toc = true
[focus]
toolchains = ['go']
+++

You want to support a number of optional custom fields for a function or type
constructor, but you need something more sophisticated than the zero value.
Perhaps you want to deliver an option that performs some _logic_ first.

<!--more-->

## Recommendation

Follow the variadic `Option` pattern!

1. Define an `Option` interface like:

   ```go
   type config struct { /* your config */ }
   type Option interface {
     apply(*config)
   }
   ```

2. Define functions that return `Option` objects:

   ```go
   func WithFoo(...) Option { ... }
   func WithBar(...) Option { ... }
   ```

3. Collect `Option`s variadically, and use the build configuration:

   ```go
   cfg := config{ /* defaults */ }
   for _, opt := range opts {
     opt.apply(&cfg)
   }
   ```

## Why

Using _variadic_ options with a sealed `interface` has a huge number of
benefits, especially when compared to an `type Options struct`-pattern:

1. You don't have to rely on the zero value for defaults. You can use semantic
   and reasonable defaults easily.

2. You can provide more than one way to assign the same value, which enables
   offering either static or dynamically-computed values

3. Since `Option` is an interface, you can make it as complex as you need --
   even returning an `error` if you want options to be capable of erroring.

4. When cleverly combined with Go's `internal` packages, you can expose
   different `Option`s in internal-facing pacakges, external packages, and even
   test-specific options for test-helper packages.

5. `Option`s can compose on top of other `Option`s. This enables powerful
   compositions, where you can add more logic to existing functions -- and allow
   clients of your library to add their own custom `Option`s.

## Example

{{< details summary="Without this pattern" >}}

Without this pattern, you might be tempted to write something like:

```go
type GitHubOptions struct {
  // Only one of 'Token' or 'CLientID'/'PEM' can be used to generate a client
  // token.

  // Token is a GitHub token
  Token string

  // ClientID is the client installation ID for a GitHub application.
  ClientID string

  // PEM is the private key for a GitHub application.
  PEM []byte
}

func NewClient(opts *GitHubOptions) (*Client, error) {
  if opts.Token != "" {
    // handle static
  } else if opts.ClientID != "" && opts.PEM != nil {
    // handle github application token
  }
  // ...
}
```

This requires branching behavior in one larger, monolithic construction
function -- all to support assigning the same underlying value: A token.

{{</ details >}}

{{< details summary="With this pattern" >}}

Using this pattern, you can create an `Option` hierarchy that define different
functions that _compute_ or determine the value statically:

```go
type config struct {
  token string
  // ...
}

type Option interface {
  apply(*config) error
}

type option func(*config) error
func (o option) apply(cfg *config) error {
  return o(cfg)
}

func StaticToken(token string) Option {
  return option(func(cfg *config) error {
    cfg.token = token;
    return nil
  })
}

func AppToken(clientID string, PEM []byte) Option {
  return option(func(cfg *config) error {
    // compute the client and PEM, return any errors
  })
}

func NewClient(opts...Option) (*Client, error) {
  cfg := &config{
    // set any defaults
  }
  // apply all options, return any errors
  for _, opt := range opts {
    if err := cfg.apply(cfg); err != nil {
      return nil, err
    }
  }
  // use the config, and return the client
}
```

{{< note >}}
With this pattern, you can now have options that _perform work_. This is a
powerful improvement over a `type Options struct` approach.
{{</ note >}}

{{</ details >}}

## Additional Notes

The approach using a {{< glossary term="sealed-interface" text="sealed" >}}
`interface` is preferred over a `func`-based definition like:

```go
type Option func(*config)
```

A `func`-based definition exposes an unnamable/unexported type, and leaks
implementation details that they don't need to know to the caller. It also
exposes the ability to `reflect` over the signature to get this type name for
the caller, which means the caller can construct your unexported type. This
leads to a **leaky abstraction**.
