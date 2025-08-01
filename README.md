<div align="center">

# treefmt-nix

<img src="https://avatars.githubusercontent.com/u/20373834" height="150"/>

**Fast and convenient multi-file formatting with Nix**

_A <a href="https://numtide.com/">numtide</a> project._

<p>
<img alt="Static Badge" src="https://img.shields.io/badge/Status-stable-green">
<a href="https://app.element.io/#/room/#home:numtide.com"><img src="https://img.shields.io/badge/Support-%23numtide-blue"/></a>
</p>

</div>

[treefmt](https://numtide.github.io/treefmt) combines file formatters for
multiple programming languages so that you can format all your project files
with a single command. With `treefmt-nix` you can specify `treefmt` build
options, dependencies and config in one place, conveniently managed by
[Nix](https://nixos.org/).

`treefmt-nix` automatically installs and configures the desired formatters as
well as `treefmt` for you and integrates nicely into your Nix development
environments. It comes with sane, pre-crafted
[formatter-configs](https://github.com/numtide/treefmt-nix/tree/main/programs)
maintained by the community; each config corresponds to a section that you would
normally add to the `treefmt` config file `treefmt.toml`.

Take a look at the already [supported formatters](#supported-programs) for
Python, Rust, Go, Haskell and more.

## Integration into Nix

### Nix classic without flakes

To run `treefmt-nix` with nix-classic, import the repo using
[`niv`](https://github.com/nmattia/niv):

```
$ niv add numtide/treefmt-nix
```

Alternatively, you can download the source and run `nix-build` in the project
root directory:

```
$ nix-build
```

The command will return the helper functions which will be later used to produce
a derivation from the specified `treefmt-nix` configuration.

After you installed treefmt-nix, specify the formatter configuration. For
instance, this one is for formatting terraform files:

```nix
# myfile.nix
{ system ? builtins.currentSystem }:
let
  nixpkgsSrc = builtins.fetchTarball "https://github.com/NixOS/nixpkgs/archive/refs/heads/nixos-unstable.tar.gz";
  treefmt-nixSrc = builtins.fetchTarball "https://github.com/numtide/treefmt-nix/archive/refs/heads/master.tar.gz";
  nixpkgs = import nixpkgsSrc { inherit system; };
  treefmt-nix = import treefmt-nixSrc;
in
treefmt-nix.mkWrapper nixpkgs {
  # Used to find the project root
  projectRootFile = ".git/config";
  # Enable the terraform formatter
  programs.terraform.enable = true;
  # Override the default package
  programs.terraform.package = nixpkgs.terraform_1;
  # Override the default settings generated by the above option
  settings.formatter.terraform.excludes = [ "hello.tf" ];
}
```

It's a good practice to place the configuration file in the project root
directory.

Next, execute this command:

```
$ nix-build myfile.nix
```

This command returns a derivation that contains a `treefmt` binary at
`./result/bin/treefmt` in your current directory. The file is actually a symlink
to the artifact in `/nix/store`.

`treefmt.toml` in this case isn't generated: the binary is wrapped with the
config.

### Flakes

Running treefmt-nix with flakes isn't hard. The library is exposed as the `lib`
attribute:

```nix
# flake.nix
{
  inputs.treefmt-nix.url = "github:numtide/treefmt-nix";
  inputs.systems.url = "github:nix-systems/default";

  outputs = { self, nixpkgs, systems, treefmt-nix }:
    let
      # Small tool to iterate over each systems
      eachSystem = f: nixpkgs.lib.genAttrs (import systems) (system: f nixpkgs.legacyPackages.${system});

      # Eval the treefmt modules from ./treefmt.nix
      treefmtEval = eachSystem (pkgs: treefmt-nix.lib.evalModule pkgs ./treefmt.nix);
    in
    {
      # for `nix fmt`
      formatter = eachSystem (pkgs: treefmtEval.${pkgs.system}.config.build.wrapper);
      # for `nix flake check`
      checks = eachSystem (pkgs: {
        formatting = treefmtEval.${pkgs.system}.config.build.check self;
      });
    };
}
```

And also add the `treefmt.nix` file (or put the content inline if you prefer):

```nix
# treefmt.nix
{ pkgs, ... }:
{
  # Used to find the project root
  projectRootFile = "flake.nix";
  # Enable the terraform formatter
  programs.terraform.enable = true;
  # Override the default package
  programs.terraform.package = pkgs.terraform_1;
  # Override the default settings generated by the above option
  settings.formatter.terraform.excludes = [ "hello.tf" ];
}
```

This file is also the place to define all the treefmt parameters like includes,
excludes and formatter options.

After specifying the flake, run
[`nix fmt`](https://nixos.org/manual/nix/stable/command-ref/new-cli/nix3-fmt.html):

```
$ nix fmt
```

Nix-fmt is a tool to format all nix files in the project, but with the specified
flake, it starts treefmt-nix and formats your project.

You can also run `nix flake check` (eg: in CI) to validate that the project's
code is properly formatted.

### Flake-parts

This flake exposes a [flake-parts](https://flake.parts/) module as well. To use
it:

1. Add `inputs.treefmt-nix.flakeModule` to the `imports` list of your
   `flake-parts` call.

2. Add `treefmt = { .. }` (containing the configuration above) to your
   `perSystem`.

As an example, see
<https://github.com/nix-community/buildbot-nix/blob/2695e33353d7bffb2073dc6a1789502dd9e7b9fd/nix/treefmt/flake-module.nix>

## Configuration

While dealing with `treefmt` outside of `nix`, the formatter configuration is
specified in a `toml` format. On the contrary, with `nix`, you write in with a
nix syntax like this:

```nix
# Used to find the project root
projectRootFile = ".git/config";
# Enable the terraform formatter
programs.terraform.enable = true;
# Override the default package
programs.terraform.package = nixpkgs.terraform_1;
# Override the default settings generated by the above option
settings.formatter.terraform.excludes = [ "hello.tf" ];
```

**Options:**

- `Project root file` is the git file of the project which you plan to format.
- The option `programs.terraform.enable` enables the needed formatter. You can
  specify as many formatter as you want. For instance:

```
programs.terraform.enable = true;
programs.gofmt.enable = true;
```

- The option `programs.terraform.package` allows you to use a particular
  build/version of the specified formatter.
- By setting`settings.formatter.terraform.excludes` you can mark the files which
  should be excluded from formatting. You can also specify other formatter
  options or includes this way.

For detailed description of the options, refer to the `treefmt`
[documentation](https://treefmt.com/latest/getting-started/configure/).

## Project structure

This repo contains a top-level `default.nix` that returns the library helper
functions.

- `mkWrapper` is the main function which wraps treefmt with the needed
  configuration.
- `mkConfigFile`
- `evalModule`
- `all-modules`

## Supported programs

<!-- `> bash ./supported-programs.sh` -->

<!-- BEGIN mdsh -->
`treefmt-nix` currently supports 111 formatters:

* [actionlint](programs/actionlint.nix)
* [alejandra](programs/alejandra.nix)
* [asmfmt](programs/asmfmt.nix)
* [autocorrect](programs/autocorrect.nix)
* [beautysh](programs/beautysh.nix)
* [biome](programs/biome.nix)
* [black](programs/black.nix)
* [buf](programs/buf.nix)
* [buildifier](programs/buildifier.nix)
* [cabal-fmt](programs/cabal-fmt.nix)
* [cabal-gild](programs/cabal-gild.nix)
* [clang-format](programs/clang-format.nix)
* [cljfmt](programs/cljfmt.nix)
* [cmake-format](programs/cmake-format.nix)
* [csharpier](programs/csharpier.nix)
* [cue](programs/cue.nix)
* [d2](programs/d2.nix)
* [dart-format](programs/dart-format.nix)
* [deadnix](programs/deadnix.nix)
* [deno](programs/deno.nix)
* [dhall](programs/dhall.nix)
* [dnscontrol](programs/dnscontrol.nix)
* [dockfmt](programs/dockfmt.nix)
* [dos2unix](programs/dos2unix.nix)
* [dprint](programs/dprint.nix)
* [efmt](programs/efmt.nix)
* [elm-format](programs/elm-format.nix)
* [erlfmt](programs/erlfmt.nix)
* [fantomas](programs/fantomas.nix)
* [fish_indent](programs/fish_indent.nix)
* [fnlfmt](programs/fnlfmt.nix)
* [formatjson5](programs/formatjson5.nix)
* [fourmolu](programs/fourmolu.nix)
* [fprettify](programs/fprettify.nix)
* [gdformat](programs/gdformat.nix)
* [genemichaels](programs/genemichaels.nix)
* [gleam](programs/gleam.nix)
* [gofmt](programs/gofmt.nix)
* [gofumpt](programs/gofumpt.nix)
* [goimports](programs/goimports.nix)
* [golines](programs/golines.nix)
* [google-java-format](programs/google-java-format.nix)
* [hclfmt](programs/hclfmt.nix)
* [hlint](programs/hlint.nix)
* [hujsonfmt](programs/hujsonfmt.nix)
* [isort](programs/isort.nix)
* [jsonfmt](programs/jsonfmt.nix)
* [jsonnet-lint](programs/jsonnet-lint.nix)
* [jsonnetfmt](programs/jsonnetfmt.nix)
* [just](programs/just.nix)
* [kdlfmt](programs/kdlfmt.nix)
* [keep-sorted](programs/keep-sorted.nix)
* [ktfmt](programs/ktfmt.nix)
* [ktlint](programs/ktlint.nix)
* [latexindent](programs/latexindent.nix)
* [leptosfmt](programs/leptosfmt.nix)
* [mdformat](programs/mdformat.nix)
* [mdsh](programs/mdsh.nix)
* [meson](programs/meson.nix)
* [mix-format](programs/mix-format.nix)
* [muon](programs/muon.nix)
* [mypy](programs/mypy.nix)
* [nickel](programs/nickel.nix)
* [nimpretty](programs/nimpretty.nix)
* [nixf-diagnose](programs/nixf-diagnose.nix)
* [nixfmt-classic](programs/nixfmt-classic.nix)
* [nixfmt-rfc-style](programs/nixfmt-rfc-style.nix)
* [nixfmt](programs/nixfmt.nix)
* [nixpkgs-fmt](programs/nixpkgs-fmt.nix)
* [ocamlformat](programs/ocamlformat.nix)
* [odinfmt](programs/odinfmt.nix)
* [opa](programs/opa.nix)
* [ormolu](programs/ormolu.nix)
* [oxipng](programs/oxipng.nix)
* [packer](programs/packer.nix)
* [perltidy](programs/perltidy.nix)
* [php-cs-fixer](programs/php-cs-fixer.nix)
* [pinact](programs/pinact.nix)
* [prettier](programs/prettier.nix)
* [protolint](programs/protolint.nix)
* [purs-tidy](programs/purs-tidy.nix)
* [rstfmt](programs/rstfmt.nix)
* [rubocop](programs/rubocop.nix)
* [ruff-check](programs/ruff-check.nix)
* [ruff-format](programs/ruff-format.nix)
* [rufo](programs/rufo.nix)
* [rustfmt](programs/rustfmt.nix)
* [scalafmt](programs/scalafmt.nix)
* [shellcheck](programs/shellcheck.nix)
* [shfmt](programs/shfmt.nix)
* [sql-formatter](programs/sql-formatter.nix)
* [sqlfluff-lint](programs/sqlfluff-lint.nix)
* [sqlfluff](programs/sqlfluff.nix)
* [sqruff](programs/sqruff.nix)
* [statix](programs/statix.nix)
* [stylish-haskell](programs/stylish-haskell.nix)
* [stylua](programs/stylua.nix)
* [swift-format](programs/swift-format.nix)
* [taplo](programs/taplo.nix)
* [templ](programs/templ.nix)
* [terraform](programs/terraform.nix)
* [texfmt](programs/texfmt.nix)
* [toml-sort](programs/toml-sort.nix)
* [typos](programs/typos.nix)
* [typstfmt](programs/typstfmt.nix)
* [typstyle](programs/typstyle.nix)
* [xmllint](programs/xmllint.nix)
* [yamlfmt](programs/yamlfmt.nix)
* [zig](programs/zig.nix)
* [zizmor](programs/zizmor.nix)
* [zprint](programs/zprint.nix)
<!-- END mdsh -->

For non-Nix users, you can also find the generated examples in the
[./examples](./examples) folder.

### Using a custom formatter

It is also possible to use custom formatters with `treefmt-nix`. For example,
the following custom formatter formats JSON files using `yq-go`:

```nix
settings.formatter = {
  "yq-json" = {
    command = "${pkgs.bash}/bin/bash";
    options = [
      "-euc"
      ''
        for file in "$@"; do
          ${lib.getExe yq-go} -i --output-format=json $file
        done
      ''
      "--" # bash swallows the second argument when using -c
    ];
    includes = [ "*.json" ];
  };
};
```

### Adding new formatters

PRs to add new formatters are welcome!

- The formatter should conform to the
  [formatter specifications](https://treefmt.com/latest/reference/formatter-spec/).
- This is not the place to debate formatting preferences. Please pick defaults
  that are standard in your community -- for instance, python is usually
  indented with 4 spaces, so don't add a python formatter with 2 spaces as the
  default.

In order to add a new formatter do the following things:

1. Create a new entry in the `./programs/` folder.
2. Consider adding yourself as the `meta.maintainer` (see below).
3. Run `./examples.sh` to update the `./examples` folder.
4. To test the program:

   1. Extend the project's `./treefmt.nix` file (temporarily) to enable the new
      formatter and configure it in whatever manner is appropriate.
   2. Add a bunch of pertinent sources in this repo -- for instance, if the new
      formatter is meant to format `*.foo` files, add a number of `*.foo` files,
      some well-formatted (and therefore expected to be exempt from modification
      by `treefmt`) and some badly-formatted.
   3. Run `nix fmt`. Confirm that well-formatted files are unchanged and that
      badly-formatted files are flagged as such. Re-run `nix fmt` and confirm
      that no additional changes were made.
   4. Add the formatter to this file [here](#supported-programs) by running:

      ```bash
      mdsh -i README.md -o README.md
      ```

      or with Nix

      ```bash
      nix run github:zimbatm/mdsh -- -i README.md -o README.md
      ```

   5. Once this is good, revert those changes.

5. Submit the PR!

### Definition of a `meta.maintainer`

You can register your desire to help with a specific formatter by adding your
GitHub handle to the module's `meta.maintainers` list.

That mostly means, for the given formatter:

- You get precedence if any decisions need to be made.
- Getting pinged if any issue is being found.

## Supported Nix versions

treefmt-nix works with all known Nix version.

If you rely on flakes and `nix fmt`, we recommend running Nix 2.25 or Lix 2.92
or later. See https://github.com/NixOS/nix/pull/11438

## Commercial support

Looking for help or customization?

Get in touch with Numtide to get a quote. We make it easy for companies to work
with Open Source projects: <https://numtide.com/contact>

## License

All the code and documentation is licensed with the MIT license.
