# Review on Azure's Go SDK

These notes are based on the branch `track2` of
`github.com/Azure/azure-sdk-for-go`.

<!-- vim-markdown-toc GFM -->

* [TL;DR;](#tldr)
* [Things that can be changed](#things-that-can-be-changed)
  * [Different naming convention for packages and package splitting](#different-naming-convention-for-packages-and-package-splitting)
  * [Add "WithContext" to function names using a context.Context](#add-withcontext-to-function-names-using-a-contextcontext)
  * [Sub-version date-versioning as semver](#sub-version-date-versioning-as-semver)
* [Things that should be changed](#things-that-should-be-changed)
  * [Improving import paths](#improving-import-paths)
  * [Simplify naming of functions, structures and variables](#simplify-naming-of-functions-structures-and-variables)
  * [Clients returned should be interfaces implemented by private structures](#clients-returned-should-be-interfaces-implemented-by-private-structures)
* [Things that must be changed](#things-that-must-be-changed)
  * [Use structures and/or functional options for passing multiple options](#use-structures-andor-functional-options-for-passing-multiple-options)
  * [Document packages and follow idiomatic-documentation patterns](#document-packages-and-follow-idiomatic-documentation-patterns)
  * [Use fmt.Sprintf or strings.Builder for string building](#use-fmtsprintf-or-stringsbuilder-for-string-building)
  * [Add tests](#add-tests)

<!-- vim-markdown-toc -->

## TL;DR;

I'm a developer. I want to use Azure's Go SDK in an idiomatic way, following
patterns I'm using already in the rest of my code.

I would love to use it as:

```golang
package main

import (
  "ctx"

  "go.azure.org/storage/blob"
  "go.azure.org/storage/container"
  "go.azure.org/azidentity/clicred"
)

func main() {
  creds, err := clicred.New()
  if err != nil {
    // handle error
  }
  
  ctx := ctx.Background()
  containerOpts := container.Options{Credentials: creds, Name: "name"}
  container, err := container.NewWithContext(ctx, containerOpts)
  // container, err := container.New(containerOpts)
  // container, err := container.Get(containerOpts)
  if err != nil {
    // handle error
  }

  cli, err := blob.New(blob.Config{Container: container, Credentials: creds})
  if err != nil {
    // handle error
  }

  err := cli.UploadWithContext(ctx, blob.UploadInput{
    Name: "helloworld",
    File: <io.Reader implementation>,
  })
  if err != nil {
    // handle error
  }
}
```

And this is great as an example for a general idea on how to use it, but the
most likely way to use it in my systems would be inside a package, so something
like uploading blobs would end up looking like this:

```golang
-- converter/converter.go --
package converter

import (
  "go.azure.org/storage/blob"
)

type Converter struct {
  blobcli blob.Blob
}

func New() (Converter, error) {
  // define here what a you need. Either pass creds or already initialized
  // instances of blob.Blob.
  return Converter{}
}
-- converter/blob.go --
package converter

import (
  "ctx"
  "io"

  "go.azure.org/storage/blob"
)

func (c *Converter) Upload(name string, f io.Reader) error {
  return c.blobcli.Upload(blob.UploadInput{
    Name: name,
    File: f,
  })
}
-- converter/blob_test.go --
package converter

import (
  "strings"
  "testing"

  "go.azure.org/storage/blob"
)

type mockBlob struct {
  blob.Blob
}

func (m mockBlob) Upload(in blob.UploadInput) error {
  return nil
}

func TestUpload(t *testing.T) {
  s := strings.NewReader("random text")
  c := &Converter{blobcli: mockBlob{}}
  err := c.Upload("filename", s)
  if err != nil {
    t.Errorf("error uploading file: %v", err)
  }
}
```

In this code `blob.Blob` is an interface and the implementation is initialized
during the `converter.New()` run. This will allow the developer to mock a simple
implementation of `blob.Blob` for testing. on the `converter/blob_test.go` a
mock test for Blob actions would look like the `mockBlob` struct

## Things that can be changed

### Different naming convention for packages and package splitting

Currently things are split in big groups: storage, compute, azidentity and it
stops there. But if the CLI structure were to follow, things would be broken
down even more, storage has files, blob, containers, etc. This would allow to
have better and simpler naming for the structures and simpler interfaces as
well.

Right now packages are structured as if `encoding` had `json`, `xml` and `html`
modules in the same directory instead of independent between themselves.

Following the structure and subdivision already made in the CLI would be great
and save so much time. It saves time and effort on the documentation as well
since the workflow would be pretty much the same.

https://docs.microsoft.com/en-us/azure/storage/scripts/storage-blobs-container-calculate-size-cli?toc=/cli/azure/toc.json

- `az storage container create` -> `/storage/container.{Create,New}`.
- `az storage blob upload-batch` -> `/storage/blob.UploadBatch`.
- `az storage blob list` -> `/storage/blob.List`.

This simplifies design, simplifies naming, simplifies documentation, simplifies
operation for the user going between CLI and Go implementations.

### Add "WithContext" to function names using a context.Context

This is in can, but should be in should. The idiomatic way of writing
functions that require `context.Context` is adding `WithContext` as a prefix to
the function's name. Is in "can" only because I'm in doubt if this is a
decision made because is expected to have context being passed in all the
functions.

### Sub-version date-versioning as semver

This is being used in other Azure Go modules: https://github.com/Azure/azure-storage-blob-go#version-table

This is a very good idea and allows to have effective versioning which matches
how Go modules are versioned. Plus, it helps users to figure out if something is
broken and to have simpler names and import paths.

## Things that should be changed

### Improving import paths

Currently import paths look like:

```golang
import (
  "github.com/Azure/azure-sdk-for-go/sdk/arm/compute/2019-12-01/armcompute"
  "github.com/Azure/azure-sdk-for-go/sdk/arm/storage/2019-06-01/armstorage"
)
```

There is `azure` twice (and with diffent capitalization), `sdk` twice, `arm`
twice, `storage` twice or `compute` twice. There's explicit versioning as well
but that is covered in another section of this document.

This can be easily solved by using vanity URLs. Uber uses it so importing `zap`
is as easy as:

```golang
import (
  "go.uber.org/zap"
)
```

The idea for this could be:

```golang
import (
  // Simplest change to apply.
  "go.azure.org/compute"
  "go.azure.org/storage"

  // If explicit versioning in the package name is wanted.
  "go.azure.org/storage/2019-06-01"

  // These last two is how it would look if the naming scheme can be
  // changed. See that section in this document for a better explanation.
  "go.azure.org/storage/container"
  "go.azure.org/storage/blob"
)
```

This does **not** requires any change in the naming scheme in the repository,
the vanity URL server will take care of this so is a low-effort change with big
gains in usability.

This is a simple implementation that GCP uses: https://github.com/GoogleCloudPlatform/govanityurls

### Simplify naming of functions, structures and variables

In the first example in the document the creation of CLI credentials is as easy
as:

```golang
// Current
creds, err := azidentity.NewAzureCLICredential(opts)
// Suggested
creds, err := clicred.New(opts)
```

Function naming should be as simple as possible and this can be helped by making
smaller and simpler packages. Just as interfaces should be smaller and simpler.

Structures like `ClientOptions` should be just `Options`.

Functions' signatures should be simpler:

```golang
// Current
ExtendImmutabilityPolicy(ctx context.Context, resourceGroupName string, accountName string, containerName string, ifMatch string, blobContainersExtendImmutabilityPolicyOptions *BlobContainersExtendImmutabilityPolicyOptions) (*ImmutabilityPolicyResponse, error)
// Suggested
ExtendImmutabilityPolicy(ctx context.Context, resourceGroup string, account string, container string, ifMatch string, policyOptions *BlobContainersExtendImmutabilityPolicyOptions) (*ImmutabilityPolicyResponse, error)
```

It would help as well to use structures here (check the respective section of
the document for a longer explanation) but it can still be used as an example of
longer than it needs variable name. The shorter the better, the names just need
to be readable and contextually valid.

### Clients returned should be interfaces implemented by private structures

The returns of methods like `azidentity.NewAzureCLICredential` should be an
interface in the signature, and an implementation of the interface by using a
private structure.

This will help users to quickly and easily take advantage of dependency
injection and mock tests quickly. This is already done in the subcommands of
`compute`, for example, like `DisksOperations`.

This can be confusing for developers if the package structure is maintained
as-is right now, because this is not idiomatic in Go.

## Things that must be changed

### Use structures and/or functional options for passing multiple options

Optimal situation would be: use functional options for creating new instances
where there are default options that could be set and structures for passing
options to function on instances. If only a method is wanted then structures is
the way to go.

Dave Cheney's has a great post on [functional options for friendly
APIs][friendlyapis]. The functions' signatures end-up being very clean and
simple, the options can be documented and added as much logic as required in a
very clean way.

    TODO how this would look like

[friendlyapis]: https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis

### Document packages and follow idiomatic-documentation patterns

If I run:

```
go doc -all ./sdk/arm/compute/2019-12-01/armcompute/ | less
```

The result is that nothing is documented as it should. There's no basic
documentation for the package, so no short description, this should look like:

```golang
// Package azidentity provides the client and types for making API requests to
// Azure's Identity service.
package azidentity
```

Every exported element should be documented and there's no need to do it as:

```golang
  // BeginCreateOrUpdate - Creates or updates...
```

The idiomatic way is:

```golang
  // BeginCreateOrUpdate starts create or updates of a container service with
  // specified configuration of orchestrator...
```

### Use fmt.Sprintf or strings.Builder for string building

This way to build strings is very weird and non-performant: [1][weirdstrings1],
[2][weirdstrings2] and so on.

[weirdstrings1]: https://github.com/Azure/azure-sdk-for-go/blob/track2/sdk/arm/storage/2019-06-01/armstorage/blobcontainers.go#L71-L87
[weirdstrings2]: https://github.com/Azure/azure-sdk-for-go/blob/track2/sdk/arm/storage/2019-06-01/armstorage/blobcontainers.go#L122-L126

Performance-wise is the worse option:

```
❯ go test -v -bench=. -benchtime 5s -benchmem
goos: darwin
goarch: amd64
pkg: github.com/nrxr/azure-review/benchmarks/stringconcat
BenchmarkJoin
BenchmarkJoin-8                         86941962                63.6 ns/op            16 B/op          1 allocs/op
BenchmarkSprintf
BenchmarkSprintf-8                      20356816               298 ns/op              96 B/op          6 allocs/op
BenchmarkConcat
BenchmarkConcat-8                       39421702               152 ns/op              32 B/op          4 allocs/op
BenchmarkConcatOneLine
BenchmarkConcatOneLine-8                99958263                57.3 ns/op             0 B/op          0 allocs/op
BenchmarkBuffer
BenchmarkBuffer-8                       80482536                71.8 ns/op            64 B/op          1 allocs/op
BenchmarkBufferWithReset
BenchmarkBufferWithReset-8              147566794               40.8 ns/op             0 B/op          0 allocs/op
BenchmarkBufferFprintf
BenchmarkBufferFprintf-8                20845557               289 ns/op              80 B/op          5 allocs/op
BenchmarkBufferStringBuilder
BenchmarkBufferStringBuilder-8          86770474                67.7 ns/op            24 B/op          2 allocs/op
BenchmarkStringsReplace
BenchmarkStringsReplace-8               14658063               407 ns/op             192 B/op         10 allocs/op
PASS
ok      github.com/nrxr/azure-review/benchmarks/stringconcat    59.381s
```

Readability-wise is the worse option as well. The best combination of
readability and performance is given by `strings.Builder`. Concat in one-line is
very easy to read as well and the most performant. The idiomatic way would be
`fmt.Sprintf`.

### Add tests

```
~/code/src/github.com/Azure/azure-sdk-for-go track2
❯ find ./sdk/ | grep "_test" | wc -l
      29

~/code/src/github.com/Azure/azure-sdk-for-go track2
❯ find ./sdk/ | wc -l
     381
```

Is not just adding tests but adding examples as well. These get added to the
documentation and helps users to pick up quickly how to implement the objects
and take advantage of the methods. It helps as well to serve as inspiration for
best approaches on implementations.

It should add benchmarks in order to make sure there's no performance
regressions in changes and that code being implemented will not add a burden to
customers' applications.
