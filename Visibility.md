# Visibility Control

(TBD)

This section will describe wgsl enhancements to control which WGSL elements are visible to importers.

* which wgsl elements are available to import from wgsl modules within the same package?
* which wgsl elements are available to import from wgsl modules in other packages?
* how to re-export elements so that they're visible with a different path or name?
* lib.wgsl to make things visible at the root of a package?
* controlling host visible names like entry points and overrides?
* Should export allow `as` renaming?
* Treat .wgsl files as .wesl with every element exported?
* Why not export struct Foo?
  * Many current wgsl parsers (including wgpu's naga) would
    choke on the unknown attribute as is and feels like having
    two export forms is a bit inconsistent.

## Export

One natural extension is to add explicit exports.
For one, this would allow library authors to hide certain functions or structs from the outside world.
It would also enable re-exports, where a library re-exports a function from another library.

There are two variations of exports, which could be combined like in Typescript

### Exporting a list of items

A standalone export statement simply specifies which globals are exported.
Then, imports can be checked against the list of exports. This is very easy to parse and implement.

```
export { foo, bar };
```

And when one wants to export everything defined in the current file, they can use the `*` syntax.

```
export *;
```

To re-export an item, one can use the same syntax.

```
import my/lighting/{ pbr };

export { pbr };
```

### Exporting as an attribute

Exports can also be added as an attribute to the item itself.

```
@export
struct Foo {
    x: f32;
}

@export
fn foo() {}
```

This is more user friendly, but also more complex to parse. It requires a partial parsing of the WGSL file to find the exports and their names.
A future export specification would include the minimal WGSL syntax that is necessary to implement this.

## Translating Source File Paths to Import Module Paths

Tools that look at source code will refer to a `package_root` in `wgsl.toml` that defines
the common prefix of `.wesl` and `.wgsl` files.

Source directories and files under the `package_root` and map directly to module paths
under the package name as expected.
e.g.

* Source file `C:\Users\lee\myProj\wgpu\foo.wesl` contains `export fn bar() {}`
* `wgpu` is the `project_root` in `wgsl.toml`.
* The project as published as package `fooz`.
* Other projects can write `import fooz/foo/bar;` to use `bar()`.

### lib.wgsl

If there is a file named `lib.wgsl` in the `package_root` directory,
any public exports in `lib.wgsl` are visible at the root of the module.
e.g. if a package ‘pkg’ has a source file `pkg_root/lib.wgsl`
that contains `export fn fun()`,
a module file in another package can import that function with `import pkg/fun`.

## An Argument for Private by Default

### Libraries Need API Control

We want library publishers to be able to decide what to expose as
part of their api. Libraries want to maintain and improve over time in a way
that’s compatible with existing users. Otherwise publishers are stuck with their
first implementation approach forever, or more likely publishers change the
implementation anyway, and then library users need to expect to edit their wgsl
code to upgrade to a new published library version. Upgrades get scarier for
publishers and consumers. That added friction makes it harder to get to a viable
wgsl library ecosystem. So we want to enable an api vs internal
distinction for libraries.
If a library publisher makes a privacy choice the user doesn’t like, the user
can still fork/edit to make things public.

### Public or Private by Default?

Libraries should have the ability to distinguish the public part
of the api (and that the language should help with that). Then the question
becomes whether public or private should be the default. A software engineering
perspective is a good way to think about the public/private question.
Software engineering perspective in the sense that the api boundary serves as a way to help
communication between multiple programmers over time. If all the functions are
fresh in one programmer’s mind, then a public/private distinction doesn’t matter
much. On the other hand, if there are multiple engineers involved, api
boundaries help clarify communication. There are two boundaries to consider: the
api to code in another package and the api to code in the same package.

#### Private by Default for Libraries

Communication between programmers working on different packages is likely quite
limited, so there’s a strong argument that strong barriers are the right
default choice. So private by default makes sense between packages. By requiring
annotations to make something public, we require the publisher to think about
their api. Good fences make good neighbors and all that.
The default choice doesn't lead to an accidental maintenance problem.
Let’s annotate with
e.g. `public` (or `pub` or `@public`) so that private can be the default, and package
publishers are pushed to define an api.

#### Private by Default Inside the Package

Communication between programmers working on the same package might be smoother
than communications between teams working on different packages. At the least,
people working on the same package are probably working in the same git
repository. Should the language help distinguish public vs private for imports
within the package? Yes, and for the same reasons as for library publishers. A
significant enough internal module wants a boundary to aid in ongoing
maintenance. So the language should support distinguishing internal from
api for within the package.

Should the default for within package visibility be public or private? For an
insignificant internal module where everything ought to be public, it’d be
overhead to have to annotate everything if private is the default. But for a
module that does want to distinguish an api, strong boundaries make sense, and
so private by default is better because it forces the writer to at least briefly
consider where the api boundary should lie. So more noise in files that don’t
care about an api, or more encouragement to define an api?

Private by default within the package makes sense as well, e.g. with an annotation
like `export` to expand vibility.  
Accidental coupling between internal modules happens all too easily even on
small teams (even one person communicating with their future self). The argument
for default private within the package is admittedly not as strong as the argument
for default private at the package boundary. But on balance
encouraging stronger internal apis is worth the cost of some syntactic noise.

Note that public outside the package could be combined with public inside the package.
So an element can be public to imports inside the package or public to imports both
both inside and outside.

### Visibility in Other Lanaguages

* JavaScript and TypeScript are private by default, annotate for public.
* Rust is private by default
but public by default for module descendants, so a bit of a hybrid.
* Go and Dart distinguish public/private by first letter rather than by annotation
  * Go leans private by default (first letter capitalize to mark as public)
  * Dart leans public by default (first letter _ to mark as private)