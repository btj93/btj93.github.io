---
title: Typed HTTP errors in Go — a pattern I keep reaching for
date: 2025-08-23
permalink: /typed-http-errors-go
---

# Typed HTTP errors in Go — a pattern I keep reaching for

Every Go service I've ever worked on grows the same chunk of code over time. It always starts as a one-liner:

```go
return c.JSON(http.StatusBadRequest, map[string]string{"error": err.Error()})
```

Then someone needs to attach a machine-readable error code. Then someone wants metadata. Then someone realizes wrapped errors leak internal detail in the response body. Then a frontend dev opens a ticket because every endpoint returns errors in a slightly different shape.

I keep ending up with the same pattern. Time to write it down before I have to invent it from scratch again.

## What I want from an HTTP error

When I introduce errors at the boundary of a Go service, here's the contract I keep wanting:

1. **A single shape.** Frontends should be able to decode every error response with the same struct.
2. **A machine-readable code.** The HTTP status alone isn't enough — `400 Bad Request` could be twenty different things. The body needs to name the thing.
3. **A safe message.** The user-facing detail should *never* leak a wrapped error. If something says `pq: duplicate key value violates unique constraint "users_email_key"`, that's an internal detail. The response should say "email already taken".
4. **Optional metadata.** A `meta` object that endpoints can fill with structured context: a field name, a retry-after, a request ID, whatever.
5. **No global mutation.** Building an error should be a pure function on the input. No `WithRequestID()` that secretly stuffs things into a package-level map.

That's the bar.

## The response shape

I always end up with something close to this:

```json
{
  "error": {
    "code": "InvalidParameter",
    "detail": "email is required",
    "meta": {
      "field": "email",
      "request_id": "01HXC9P4ABCDE..."
    }
  }
}
```

`code` is a short PascalCase enum value. `detail` is a sentence I'd be okay showing to the user. `meta` is an open-ended object — but every field in it is *added on purpose*. No "throw the wrapped error in there and let the frontend figure it out".

## The Go side

The interface I want to write at a handler looks like this:

```go
if email == "" {
    return httperror.UnprocessableEntity(
        ErrCodeInvalidParameter,
        ErrEmailRequired,
    ).WithField("field", "email")
}
```

To make that work, I need a builder type that:

- carries an HTTP status,
- carries a code,
- wraps an underlying `error`,
- and exposes immutable `WithField` / `WithFields` chaining.

A first cut:

```go
package httperror

import (
    "encoding/json"
    "errors"
    "net/http"
)

type ErrCode string

type Error struct {
    Status  int
    Code    ErrCode
    Err     error            // wrapped, surfaced only via errors.Is/As
    Meta    map[string]any   // user-facing metadata only
}

func (e *Error) Error() string {
    return string(e.Code) + ": " + e.Err.Error()
}

func (e *Error) Unwrap() error { return e.Err }

// MarshalJSON enforces the safe response shape.
// Notably, the wrapped error's full chain is NOT exposed —
// only its top-level message.
func (e *Error) MarshalJSON() ([]byte, error) {
    return json.Marshal(struct {
        Error struct {
            Code   ErrCode        `json:"code"`
            Detail string         `json:"detail"`
            Meta   map[string]any `json:"meta,omitempty"`
        } `json:"error"`
    }{
        Error: struct {
            Code   ErrCode        `json:"code"`
            Detail string         `json:"detail"`
            Meta   map[string]any `json:"meta,omitempty"`
        }{
            Code:   e.Code,
            Detail: e.Err.Error(), // top-level only
            Meta:   e.Meta,
        },
    })
}
```

The `Detail` field deliberately uses `e.Err.Error()` and not the *full* chain via `%+v`. Wrapped causes stay reachable on the server side via [`errors.Unwrap`](https://pkg.go.dev/errors#Unwrap), but they don't leak to the client.

## The chainable constructors

Status-code constructors are nice because they read like English:

```go
func BadRequest(code ErrCode, err error) *Error {
    return &Error{Status: http.StatusBadRequest, Code: code, Err: err}
}

func UnprocessableEntity(code ErrCode, err error) *Error {
    return &Error{Status: http.StatusUnprocessableEntity, Code: code, Err: err}
}

func InternalServerError(code ErrCode, err error) *Error {
    return &Error{Status: http.StatusInternalServerError, Code: code, Err: err}
}
```

`WithField` should return a new error rather than mutating:

```go
func (e *Error) WithField(key string, value any) *Error {
    cloned := *e
    if cloned.Meta == nil {
        cloned.Meta = map[string]any{}
    } else {
        cloned.Meta = cloneMap(cloned.Meta)
    }
    cloned.Meta[key] = value
    return &cloned
}
```

Why immutable? Because the alternative — mutating in place — surprises you the moment you reuse a base error:

```go
var ErrInvalidParam = httperror.BadRequest(ErrCodeInvalidParam, errors.New("invalid"))

func handler1(...) error {
    return ErrInvalidParam.WithField("field", "email") // mutates ErrInvalidParam!
}

func handler2(...) error {
    return ErrInvalidParam.WithField("field", "age") // sees previous field
}
```

I've shipped that bug. Once was enough.

## Wiring it into Echo (or any framework)

The framework-specific bit is dead simple. For [Echo](https://echo.labstack.com/), a single middleware that knows about your error type:

```go
func ErrorHandler(err error, c echo.Context) {
    var httpErr *httperror.Error
    if errors.As(err, &httpErr) {
        _ = c.JSON(httpErr.Status, httpErr)
        return
    }
    // unknown error: fall back to opaque 500
    _ = c.JSON(http.StatusInternalServerError, httperror.InternalServerError(
        "InternalServerError",
        errors.New("internal server error"),
    ))
}
```

Critically, the framework integration is *separate* from the core type. The core `Error` doesn't import `echo` or `gin` or `chi`. That way you can swap frameworks without rewriting your error vocabulary.

A pattern I've seen go wrong is when the error library directly returns `echo.HTTPError`, coupling the entire codebase to one framework. Don't. Keep the error type pure data; let an adapter translate.

## Why context-injection is tempting and dangerous

A feature I keep flirting with is auto-injecting context values:

```go
err := httperror.WithContext(ctx, []ContextKey{RequestID{}}).
    BadRequest(ErrCodeInvalidParam, ErrInvalidInput)
```

It's tempting because every endpoint wants the request ID in the response. But two warnings from experience:

1. **The context value is snapshotted at construction time.** If you build the error early and return it late, and the request ID changed somewhere in between, you'll be very confused. Solve this with documentation, not cleverness.
2. **Don't auto-include anything that might be PII.** Don't have a "well let's just dump the entire ctx" mode. The whole point of typed errors is that the response body is intentional.

If I include `WithContext`, I make the allowed keys *explicit*: each key implements an interface like `ContextKey { Key() string }`. You can't accidentally include something you didn't authorize.

## Lessons after several iterations

A few things I keep coming back to:

- **Define the response schema first.** The Go API should bend to fit the JSON contract, not the other way around.
- **Wrap, don't replace, the underlying error.** Use `Unwrap` so `errors.Is`/`errors.As` keep working.
- **Don't expose the unwrap chain to the client.** It's a debugging aid, not a public API.
- **Make the builder immutable.** Shared base errors are too convenient to give up; chained mutation will bite.
- **Keep framework integration in a sub-package.** The error type should be pure data.
- **Resist the temptation to add a "details" string field that's "just for the user".** You'll have one place that formats and one that doesn't, and you'll never be sure which.

The whole thing is maybe 200 lines of Go, but it's 200 lines I end up writing on every greenfield service. Better to write them once, on purpose, than to grow them by accretion.

Happy erroring!

## References

- Go `errors` package — [`pkg.go.dev/errors`](https://pkg.go.dev/errors) (covers `Is`, `As`, `Unwrap`, `Join`)
- `fmt.Errorf` and the `%w` verb — [`pkg.go.dev/fmt#Errorf`](https://pkg.go.dev/fmt#Errorf)
- Go `encoding/json` package — [`pkg.go.dev/encoding/json`](https://pkg.go.dev/encoding/json) (custom `MarshalJSON`)
- Go `net/http` status code constants — [`pkg.go.dev/net/http`](https://pkg.go.dev/net/http)
- Echo web framework — [`echo.labstack.com`](https://echo.labstack.com/)
