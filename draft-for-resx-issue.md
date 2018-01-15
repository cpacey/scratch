# RFC: Design for compiling resx via resgen

## Background

C# DLLs can have embedded resources (files that are stuffed into the DLL). In msbuild/csproj when you embed a resources with file type .resx it is first passed to `resgen.exe` along with all the referenced DLLs to be compiled into a .resource file (which is then embedded in the DLL).

Currently, the rules_dotnet rules accept `.resx` files (along with `.cs` files) in the `srcs` attribute of `csharp_library` (etc.) but don't specially handle them (my testing indicates if you pass a resx file to the rules the compile will actually fail).

In PR #50 I added support for resources via a `resource` attribute.

I'd like to support emitting actions to call `resgen.exe` but I'm not sure what the best/idiomatic design for the rules would look like. I've outlined two major options below:

## Possible APIs

### (A) All in one magic

```python
csharp_library(
  name = "mylib",
  srcs = [ "foo.cs" ],
  resources = [
    "foo.bmp",
    "foo.resx"
  ],
  deps = [
    "//othercsharplibs:A",
    "//othercsharplibs:B",
  ],
)
```

**How this works**: everything in `resources` that isn't a .resx file is passed(?) to `csc.exe` directly with `/resource:%s`. .resx files are first passed (in a separate action) to `resgen.exe` along with `deps` to generate a .resource file which is passed via `/resource:%s`.

**Why this is nice**: It just works™: developers probably don't want to know the gory details here. This matches the behavior of msbuild+csproj where msbuild figures out that it needs to call resgen for you (.resx files look the same as vanilla resources inside a csproj).

**A variation**: Include resx files in `srcs`: this is maybe more explicit. .resx files have to be compiled first. If this option were chosen it might be nice to ban .resx from `resources` to avoid confusion. .resx is currently allowed in `srcs` so maybe that was the intent.

```python
csharp_library(
  name = "mylib",
  srcs = [
    "foo.cs"
    "foo.resx"
  ],
  resources = [ "foo.bmp" ],
  deps = [
    "//othercsharplibs:A",
    "//othercsharplibs:B",
  ],
)
```

In either case, the actions look like:

```python
ctx.actions.run(
  inputs = ["foo.resx", "//othersharplibs:A.dll", "//othercsharplibs:B.dll" ],
  outputs = [ "foo.resource" ],
  executable = "resgen.exe",
  ...
)

ctx.actions.run(
  inputs = [ "foo.cs", "foo.bmp", "foo.resource", "//othercsharplibs:A.dll", "//othercsharplibs:B.dll", ],
  outputs = "mylib.dll",
  executable = "csc.exe",
  ...
)
```

### (B) Separate `resource_library` rule:

```python
resource_library(
  name = "rlib",
  srcs = [
    "foo.bmp",
    "bar.resx",
  ],
  deps = [ "//othercsharplibs:A" ],
)

csharp_library(
  name = "mylib",
  srcs = [ "foo.cs" ],
  resources = [ "other.txt" ],
  deps = [
    "//othercsharplibs:B",
    ":resourcelib",
  ],
)
```

**How this works**: `rlib` has an action to call `resgen.exe` on `bar.resx` to generate `bar.resource`. When `mylib` depends on `:resourcelib` it picks up `foo.bmp` and `bar.resource` which get passed with `other.txt` to `csc.exe` via a `/resource:%s` argument. `mylib` also picks up the transitive dep on `//othercsharplibs:A`.

**Why this is nice**: the `deps` on `rlib` can be explicitly listed, rather than using all `deps` from `mylib`. This would improve caching and make it easier to share compiled `resx` resources between different DLLs. It is less magic.

The actions look something like this (note that the `resgen.exe` action has fewer inputs):

```python
ctx.actions.run(
  inputs = ["foo.resx", "//othersharplibs:A.dll" ],
  outputs = [ "foo.resource" ],
  executable = "resgen.exe",
  ...
)

ctx.actions.run(
  inputs = [ "foo.cs", "foo.bmp", "other.txt", "foo.resource", "//othercsharplibs:A.dll", "//othercsharplibs:B.dll", ],
  outputs = "mylib.dll",
  executable = "csc.exe",
  ...
)
```

### (C) Both

Support (A) and (B) at the `BUILD` authors choice.

## My thoughts

I'm slightly inclined to (C), starting with (A) first implementation-wise. At the action level there isn't much difference (other than (B) allowing more control (but the benefits may be minimal.)) I don't have strong opinions though.
