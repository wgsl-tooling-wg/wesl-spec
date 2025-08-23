# Glossary of WGSL Importing terms
* **WESL** - The extended WGSL language, and is pronounced like "weasel". Stands for WGSL Extended Shading Language
* **WESL translator** - a program that implements the WESL specification
  and transpiles WESL source code to WGSL.
* **WESL linker** - a WESL translator.
  Part of the process of translating WESL involves
  linking together multiple WGSL and WESL modules.
  Also akin to **bundler** in the JavaScript/TypeScript world.
* **Importable item**
  * Structs
  * Functions
  * Type aliases
  * [Const declarations, override declarations](https://www.w3.org/TR/WGSL/#value-decls)
  * [Var declarations](https://www.w3.org/TR/WGSL/#var-decls)
* **Module**: A single WESL or WGSL file.
* **Root Module**: The WESL module from which translation starts. A single project can have many root modules.
* **Module Path**: Hierarchical address of a module file or partial path, akin to a filesystem path
* **Side effects**: WESL/WGSL shader code that is visible to host code (e.g. in Rust or JavaScript).
  Changes to that shader code have the side effect of changing the host interface to the shader.
  * Things that are specified when [creating a WGSL pipeline](https://developer.mozilla.org/en-US/docs/Web/API/GPUDevice/createRenderPipeline#fragment_object_structure)
    * Shader entry-points
    * Pipeline-overridable constants
    * Global variables, including bindings
  * [Directives](https://www.w3.org/TR/WGSL/#directives): Generated WGSL code must agree on a set of directives
  * (Maybe `const_assert`?)
* **Package**: A publishable body of WESL code containing multiple files. Akin to a JavaScript npm package or a Rust crate.
* **Package Root**: The root directory containing WESL files
* **Visibility**: Whether a importable item is visible to
  * other modules
  * other packages
