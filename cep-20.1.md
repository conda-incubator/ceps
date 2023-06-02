# A new recipe format – part 1

<table>
<tr><td> Title </td><td> A new recipe format </td>
<tr><td> Status </td><td> Proposed</td></tr>
<tr><td> Author(s) </td><td> Wolf Vollprecht &lt;wolf@prefix.dev&gt;</td></tr>
<tr><td> Created </td><td> May 23, 2023</td></tr>
<tr><td> Updated </td><td> May 23, 2023</td></tr>
<tr><td> Discussion </td><td>  </td></tr>
<tr><td> Implementation </td><td>https://github.com/prefix-dev/rattler-build</td></tr>
</table>

## Abstract

We propose a new recipe format that is heavily inspired by conda-build. The main
change is a pure YAML format without arbitrary Jinja or comments with semantic
meaning.

## Motivation

The conda-build format has grown over the years to become quite complex.
Unfortunately it has never been formally "specified" and it has grown some
features over time that make it hard to parse as straightforward YAML.

The CEP attempts to introduce a subset of the conda build format that allows for
fast parsing and building of recipes.

### History

A discussion was started on what a new recipe spec could or should look like.
The fragments of this discussion can be found here:
https://github.com/mamba-org/conda-specs/blob/master/proposed_specs/recipe.md
The reason for a new spec are:

- Make it easier to parse ("pure yaml"). conda-build uses a mix of comments and
  jinja to achieve a great deal of flexibility, but it's hard to parse the
  recipe with a computer
- iron out some inconsistencies around multiple outputs (build vs. build/script
  and more)
- remove any need for recursive parsing & solving

## Major differences with conda-build

- no full Jinja2 support: no conditional or `{% set ...` support, only string
  interpolation. Variables can be set in the toplevel "context" which is valid
  YAML
- Jinja string interpolation needs to be quoted at the beginning of a string,
  e.g. `- "{{ version }}"` in order for it to be valid YAML
- Selectors use a YAML dictionary style (vs. comments in conda-build). E.g. `-
  sel(osx): somepkg` instead of `- somepkg  # [osx]`

## Selectors

Selectors in the new spec take the following format:

`sel(unix): selected_value`

This is a valid YAML dictionary. Selector contents are simple boolean
expressions and follow Python syntax. The following selectors are all valid:

```
win and arm64
(osx or linux) and aarch64
something == "test"
```

### The cmp function for variant selection

Furthermore, we have a special "cmp" function that can be used to run a check
against a selected variant version. The `cmp` function looks like the following:

```
cmp(python, "3.6")
cmp(python, ">=3.6")
cmp(python, ">=3.8,<3.10")
etc
```

This can be used in a selector like so:

```
requirements:
  build:
    - sel(cmp(python, ">=3.6,<3.10")): dataclasses
```

This functionality generalizes and replaces the previous special variables such
as `py2k`, `py3k`, `py36`, `py37`, and works just as well for NumPy, Ruby, R, or
any other variant that might be of interest in the future.

### Preprocessing selectors

You can add selectors to any item, and the selector is evaluated in a
preprocessing stage. If a selector evaluates to `true`, the item is flattened
into the parent element. If a selector evaluates to `false`, the item is
removed.

```yaml
source:
  - sel(not win):
      url: http://path/to/unix/source
  - sel(win):
      url: http://path/to/windows/source
```

Because the selector is a valid Jinja expression, complicated logic is possible:

```yaml
source:
  - sel(win):
      url: http://path/to/windows/source
  - sel(unix and cmp(python, "2")):
      url: http://path/to/python2/unix/source
  - sel(unix and cmp(python, "3")):
      url: http://path/to/python3/unix/source
```

Lists are automatically "merged" upwards, so it is possible to group multiple
items under a single selector:

```yaml
test:
  commands:
    - sel(unix):
      - test -d ${PREFIX}/include/xtensor
      - test -f ${PREFIX}/include/xtensor/xarray.hpp
      - test -f ${PREFIX}/lib/cmake/xtensor/xtensorConfig.cmake
      - test -f ${PREFIX}/lib/cmake/xtensor/xtensorConfigVersion.cmake
    - sel(win):
      - if not exist %LIBRARY_PREFIX%\include\xtensor\xarray.hpp (exit 1)
      - if not exist %LIBRARY_PREFIX%\lib\cmake\xtensor\xtensorConfig.cmake (exit 1)
      - if not exist %LIBRARY_PREFIX%\lib\cmake\xtensor\xtensorConfigVersion.cmake (exit 1)

# On unix this is rendered to:
test:
  commands:
    - test -d ${PREFIX}/include/xtensor
    - test -f ${PREFIX}/include/xtensor/xarray.hpp
    - test -f ${PREFIX}/lib/cmake/xtensor/xtensorConfig.cmake
    - test -f ${PREFIX}/lib/cmake/xtensor/xtensorConfigVersion.cmake
```

## Templating with Jinja

The spec supports simple Jinja templating in the `recipe.yaml` file.

You can set up Jinja variables in the context YAML section:

```yaml
context:
  name: "test"
  version: "5.1.2"
  major_version: "{{ version.split('.')[0] }}"
```

Later in your `recipe.yaml` you can use these values in string interpolation
with Jinja. For example:

```yaml
source:
  url: https://github.com/mamba-org/{{ name }}/v{{ version }}.tar.gz
```

Jinja has built-in support for some common string manipulations.

In the new spec, complex Jinja is completely disallowed as we try to produce
YAML that is valid at all times. So you should not use any `{% if ... %}` or
similar Jinja constructs that produce invalid yaml. Furthermore, quotes need to
be applied when starting a value with double-curly brackets like so:

```yaml
package:
  name: {{ name }}   # WRONG: invalid yaml
  name: "{{ name }}" # correct
```

Some Jinja functions remain valid, but this is out of the scope of this spec.
However, as an example, the `compiler` Jinja function will still be available,
with the main difference being the quoting of the brackets.

```yaml
requirements:
  build:
    - "{{ compiler('cxx') }}"
```

## Shortcomings

Since we deliberately limit the amount of "Jinja" that is allowed in recipes
there will be several shortcomings.

For example, using a `{% for ... %}` loop is prohibited. After searching through
the conda-forge recipes with the Github search, we found for loops mostly used
in tests. 

In our view, for loops are a nice helper, but not necessary for many tasks: the
same functionality could be achieved in a testing script, for example. At the
same time we also plan to formalize a more powerful testing harness ([prototyped
in
boa](https://github.com/mamba-org/boa/blob/main/docs/source/recipe_spec.md#check-for-file-existence-in-the-final-package)).

This could be used instead of a for loop to check the existence of shared
libraries or header files cross-platform (instead of relying on Jinja templates
as done
[here](https://github.com/conda-forge/opencv-feedstock/blob/2fc7848655ca65419050336fe38fcfd87bec0649/recipe/meta.yaml#L131)
or
[here](https://github.com/conda-forge/boost-cpp-feedstock/blob/699cfb6ec3488da8586833b1500b69133f052b6f/recipe/meta.yaml#L53)).

Other uses of `for` loops should be relatively easy to refactor, such as
[here](https://github.com/conda-forge/libiio-feedstock/blob/1351e5846b753e4ee85624acf3a14aee4bcf321d/recipe/meta.yaml#L45-L51).

However, since the new recipe format is "pure YAML" it is very easy to create
and pre-process these files using a script, or even generating them with Python
or any other scripting language. That means, many of the features that are
currently done with Jinja could be done with a simple pre-processing step in the
future.

## Examples

### xtensor

Original recipe [found
here](https://github.com/conda-forge/xtensor-feedstock/blob/feaa4fd8015f96038168a9d67d69eaf06a36d63f/recipe/meta.yaml).

```yaml
context:
  name: xtensor
  version: 0.24.6
  sha256: f87259b51aabafdd1183947747edfff4cff75d55375334f2e81cee6dc68ef655

package:
  name: "{{ name|lower }}"
  version: "{{ version }}"

source:
  fn: "{{ name }}-{{ version }}.tar.gz"
  url: https://github.com/xtensor-stack/xtensor/archive/{{ version }}.tar.gz
  sha256: "{{ sha256 }}"

build:
  number: 0
  sel(win and vc<14):
    skip: true

requirements:
  build:
    - "{{ compiler('cxx') }}"
    - cmake
    - sel(unix): make
  host:
    - xtl >=0.7,<0.8
  run:
    - xtl >=0.7,<0.8
  run_constrained:
    - xsimd >=8.0.3,<10

test:
  commands:
    sel(unix):
      - test -d ${PREFIX}/include/xtensor
      - test -f ${PREFIX}/include/xtensor/xarray.hpp
      - test -f ${PREFIX}/share/cmake/xtensor/xtensorConfig.cmake
      - test -f ${PREFIX}/share/cmake/xtensor/xtensorConfigVersion.cmake
    sel(win):
      - if not exist %LIBRARY_PREFIX%\include\xtensor\xarray.hpp (exit 1)
      - if not exist %LIBRARY_PREFIX%\share\cmake\xtensor\xtensorConfig.cmake (exit 1)
      - if not exist %LIBRARY_PREFIX%\share\cmake\xtensor\xtensorConfigVersion.cmake (exit 1)

about:
  home: https://github.com/xtensor-stack/xtensor
  license: BSD-3-Clause
  license_family: BSD
  license_file: LICENSE
  summary: The C++ tensor algebra library
  description: Multi dimensional arrays with broadcasting and lazy computing
  doc_url: https://xtensor.readthedocs.io
  dev_url: https://github.com/xtensor-stack/xtensor

extra:
  recipe-maintainers:
    - SylvainCorlay
    - JohanMabille
    - wolfv
    - davidbrochart
```

