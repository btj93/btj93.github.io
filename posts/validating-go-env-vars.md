---
title: Validating Go env vars at startup — one struct, one fail
date: 2025-11-15
permalink: /validating-go-env-vars
---

# Validating Go env vars at startup — one struct, one fail

There's a particular flavor of production bug that I find especially upsetting, and it goes like this:

1. A service deploys.
2. Health check returns 200 — the HTTP server is up.
3. A few hours later, a user does the one operation that needs `S3_BUCKET` to be set.
4. `os.Getenv("S3_BUCKET")` returns `""`.
5. The service hands back an opaque 500.

The whole class of bug is preventable, and it doesn't take much. You just have to do the work *at startup*, not lazily, and you have to make missing values *loud*.

This post is about the patterns I use in Go services to do that, and the small library shape I keep reaching for.

## The shape I want

The interface I'm aiming for at the top of `main.go` is something like this:

```go
type Config struct {
    HTTPPort   int    `env:"HTTP_PORT" default:"8080"`
    DBURL      string `env:"DB_URL" required:"true"`
    DBMaxConns int    `env:"DB_MAX_CONNS" default:"10"`
    Debug      bool   `env:"DEBUG" default:"false"`
    S3Bucket   string `env:"S3_BUCKET" required:"true"`
}

var cfg Config
if err := envalid.LoadInto(&cfg); err != nil {
    log.Fatal(err)
}
```

The contract I want from `LoadInto`:

1. **Required fields fail fast.** If `DB_URL` isn't set, the process should refuse to start. Not "log a warning", not "return zero value". Exit.
2. **Defaults are explicit.** If a field has `default:"8080"`, the zero value coming out of the env should be `8080`, not `0`.
3. **Type coercion is built in.** `os.Getenv` returns a string. I want my `int` to be an int, my `bool` to be a bool, my `time.Duration` to be a duration.
4. **Errors enumerate, not first-fail.** If three variables are missing, tell me all three. Don't make me restart and find out about the second one only after I fix the first.
5. **The list of variables is discoverable.** Some component — usually a healthcheck or `--help` — should be able to enumerate every variable the service expects.

That's the bar. Let's build it.

## A first cut

```go
package envalid

import (
    "errors"
    "fmt"
    "os"
    "reflect"
    "strconv"
    "time"
)

type FieldError struct {
    Name    string
    Reason  string
}

func (e *FieldError) Error() string {
    return fmt.Sprintf("env %s: %s", e.Name, e.Reason)
}

func LoadInto(dst any) error {
    v := reflect.ValueOf(dst).Elem()
    t := v.Type()

    var errs []error

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        name := field.Tag.Get("env")
        if name == "" {
            continue
        }

        raw, ok := os.LookupEnv(name)
        if !ok {
            if def, hasDefault := field.Tag.Lookup("default"); hasDefault {
                raw = def
            } else if field.Tag.Get("required") == "true" {
                errs = append(errs, &FieldError{Name: name, Reason: "required but not set"})
                continue
            } else {
                continue // not required, no default — leave zero value
            }
        }

        if err := assign(v.Field(i), raw); err != nil {
            errs = append(errs, &FieldError{Name: name, Reason: err.Error()})
        }
    }

    if len(errs) > 0 {
        return errors.Join(errs...)
    }
    return nil
}

func assign(v reflect.Value, raw string) error {
    switch v.Kind() {
    case reflect.String:
        v.SetString(raw)
    case reflect.Int, reflect.Int64:
        // special-case time.Duration before generic int
        if v.Type() == reflect.TypeOf(time.Duration(0)) {
            d, err := time.ParseDuration(raw)
            if err != nil {
                return err
            }
            v.SetInt(int64(d))
            return nil
        }
        n, err := strconv.ParseInt(raw, 10, 64)
        if err != nil {
            return err
        }
        v.SetInt(n)
    case reflect.Bool:
        b, err := strconv.ParseBool(raw)
        if err != nil {
            return err
        }
        v.SetBool(b)
    default:
        return fmt.Errorf("unsupported field kind: %s", v.Kind())
    }
    return nil
}
```

Less than a hundred lines and you've got 80% of what you need. The remaining 20% is taste.

## Decision: should missing vars panic or just collect?

The earlier draft of this had two entry points: `MustLoadInto` (panic on any failure) and `LoadInto` (return an error).

I went back and forth on this. The risk with "just return an error" is that someone will write:

```go
_ = envalid.LoadInto(&cfg) // YOLO
```

…and you're back to lazy failures.

The compromise I like best:

- The library exposes both `LoadInto` (returns error) and `MustLoadInto` (calls `os.Exit(1)` with a formatted message).
- Documentation strongly recommends `MustLoadInto` for application bootstrap.
- `LoadInto` exists because tests want it.

For the exit path: don't `panic`. `panic` prints a stack trace and tons of noise. `os.Exit(1)` after a clean log line is a much better operator experience. Print the missing env vars as a tidy list:

```
$ ./my-service
2025-11-15T11:00:00Z FATAL missing required environment variables:
  - DB_URL
  - S3_BUCKET
exit status 1
```

That message takes the operator from "the service exited" to "I know exactly which env block in the deploy config is broken" in one step.

## Decision: struct tag vs. registry

Two API shapes I bounced between:

```go
// (a) struct-based, declarative
type Config struct {
    DBURL string `env:"DB_URL" required:"true"`
}

// (b) registry-based, imperative
envalid.Required("DB_URL")
envalid.Optional("HTTP_PORT", "8080")
```

I've shipped both at different points. Conclusions:

- The **struct shape** is much nicer for application code. You get autocomplete, type safety, and a place where every env var is documented.
- The **registry shape** is useful when "the set of env vars" needs to be available outside of a single struct — for example, a healthcheck endpoint that lists every var the service consumes. You can derive a registry from the struct via reflection, but the inverse is messier.

So: ship both, but optimize the struct path for ergonomics, and let the registry derive from it.

```go
// derive a flat list of expected env names from a struct
func KeysOf(dst any) []string { ... }
```

The healthcheck calls `KeysOf(cfg)` and ranges over the result. New variables in the struct automatically show up.

## Decision: how to encode defaults

The `default:"8080"` tag is the simplest thing that works. But there are gotchas:

- **Struct field zero values vs. tag-specified defaults.** If a user writes `cfg.HTTPPort = 8443` *before* calling `LoadInto`, what should happen? My take: the env var (or its default) wins. The reflection-based loader has no idea what the field's pre-call value was, and trying to detect "explicit override" is a rabbit hole.
- **Empty string is a valid value.** `default:""` should *not* be the same as "no default". The library distinguishes via `field.Tag.Lookup("default")` (returns a second `ok` bool), not `Get("default") == ""`. Subtle but important.
- **Multiline defaults are evil.** I don't support them. If you have a PEM key in a default, you're doing something wrong. The default should be the simple thing, and the prod value the complex thing.

## The healthcheck wiring

Once the registry shape exists, the healthcheck integration is trivial:

```go
http.HandleFunc("/healthz/env", func(w http.ResponseWriter, r *http.Request) {
    missing := []string{}
    for _, key := range envalid.KeysOf(&cfg) {
        if os.Getenv(key) == "" {
            missing = append(missing, key)
        }
    }
    if len(missing) == 0 {
        w.WriteHeader(200)
        _, _ = w.Write([]byte(`{"ok": true}`))
        return
    }
    w.WriteHeader(500)
    _ = json.NewEncoder(w).Encode(map[string]any{
        "ok":      false,
        "missing": missing,
    })
})
```

In practice the service would never have started if anything were missing — `MustLoadInto` would have killed it. But this endpoint is useful for *post-hoc* drift detection, where someone rotated a secret without restarting the service. (Don't laugh. It happens.)

## What I do *not* try to do

A few things this kind of library tempts you to add. I keep saying no:

- **Hot reload.** "Watch the env and reload on change" is an enormous footgun. The right answer is "restart the service to pick up new config", and a restart is so cheap nowadays that you almost never need to make it hot.
- **Encrypted secrets.** Not the library's job. Decrypt in your platform layer (KMS, sealed secrets, whatever) and inject the plaintext into the environment.
- **Hierarchical merging.** YAML + env + flags + Consul + sneezing on the keyboard. Each layer of fallback doubles the surface area of "where does this value actually come from?". I would rather have one config source per environment and a clear error when it's wrong.

## Lessons

A handful that I keep relearning:

1. **Fail at startup. Loudly.** A service that comes up with bad config is a service waiting to embarrass you.
2. **Collect every error before reporting.** Don't make the operator restart five times to find five bugs.
3. **Make defaults explicit, not implicit.** "Zero value of `int` is `0`" is not a sane default for `HTTP_PORT`.
4. **Distinguish "not set" from "set to empty string".** They are different.
5. **Expose the list.** A healthcheck or `--help` flag that enumerates expected variables is worth its weight in gold.

The whole pattern is small. The bugs it prevents are big. Worth the 150 lines.

Happy starting up!

## References

- `os.LookupEnv` — [`pkg.go.dev/os#LookupEnv`](https://pkg.go.dev/os#LookupEnv)
- `reflect` package and `StructTag` — [`pkg.go.dev/reflect`](https://pkg.go.dev/reflect), [`pkg.go.dev/reflect#StructTag`](https://pkg.go.dev/reflect#StructTag)
- `strconv` for type coercion — [`pkg.go.dev/strconv`](https://pkg.go.dev/strconv)
- `time.ParseDuration` — [`pkg.go.dev/time#ParseDuration`](https://pkg.go.dev/time#ParseDuration)
- `errors.Join` — [`pkg.go.dev/errors#Join`](https://pkg.go.dev/errors#Join)
- `os.Exit` — [`pkg.go.dev/os#Exit`](https://pkg.go.dev/os#Exit)
