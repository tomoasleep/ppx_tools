ppx_tools
=========

Tools for authors of syntactic tools (such as ppx rewriters).

This package is licensed by LexiFi under the terms of the MIT license.

The tools are installed as a findlib package called 'ppx_tools', and
are thus accessible through the ocamlfind driver (e.g.: ocamlfind
ppx_tools/dumpast.exe).


dumpast.exe
-----------

This tool parses fragments of OCaml code (or entire source files) and
dump the resulting internal Parsetree representation.  Intended uses:

 - Help to learn about the OCaml Parsetree structure and how it
   corresponds to OCaml source syntax.

 - Create fragments of Parsetree to be copy-pasted into the source
   code of syntax-manipulating programs (such as ppx rewriters).

Usage:
 
  ocamlfind ppx_tools/dumpast.exe -e "1 + 2"

The tool can be used to show the Parsetree representation of small
fragments of syntax passed on the command line (-e for expressions, -p
for patterns, -t for type expressions) or for entire .ml/mli files.
The standard -pp and -ppx options are supported, but only applied on
whole files.  The tool has further option to control how location and
attribute fields in the Parsetree should be displayed.


ppx_metaquot.exe
----------------

A ppx filter to help writing programs which manipulate the Parsetree,
by allowing the programmer to use concrete syntax for expressions
creating Parsetree fragments and patterns deconstructing Parsetree
fragments.  See the top of ppx_metaquot.ml for a description of the
supported extensions.

Usage:

  ocamlfind -c -ppx "ocamlfind ppx_tools/ppx_metaquot.exe" my_ppx_code.ml



genlifter.exe
-------------

This tool generates a virtual "lifter" class for one or several OCaml
type constructors.  It does so by loading the .cmi files which define
those types.  The generated lifter class exposes one method to "reify"
type constructors passed on the command-line and other type
constructors accessible from them.  The class is parametrized over the
target type of the reification, and it must provide method to deal
with basic types (int, string, char, int32, int64, nativeint) and data
type builders (record, constr, tuple, list, array).  As an example,
calling:

    ocamlfind ppx_tools/genlifter.exe -I +compiler-libs Location.t

produces the following class:

    class virtual ['res] lifter =
      object (this)
        method lift_Location_t : Location.t -> 'res=
          fun
            { Location.loc_start = loc_start; Location.loc_end = loc_end;
              Location.loc_ghost = loc_ghost }
             ->
            this#record "Location.t"
              [("loc_start", (this#lift_Lexing_position loc_start));
              ("loc_end", (this#lift_Lexing_position loc_end));
              ("loc_ghost", (this#lift_bool loc_ghost))]
        method lift_bool : bool -> 'res=
          function
          | false  -> this#constr "bool" ("false", [])
          | true  -> this#constr "bool" ("true", [])
        method lift_Lexing_position : Lexing.position -> 'res=
          fun
            { Lexing.pos_fname = pos_fname; Lexing.pos_lnum = pos_lnum;
              Lexing.pos_bol = pos_bol; Lexing.pos_cnum = pos_cnum }
             ->
            this#record "Lexing.position"
              [("pos_fname", (this#string pos_fname));
              ("pos_lnum", (this#int pos_lnum));
              ("pos_bol", (this#int pos_bol));
              ("pos_cnum", (this#int pos_cnum))]
      end
    
dumpast.exe is a direct example of using genlifter.exe applied on the
OCaml Parsetree definition itself.  ppx_metaquot.exe is another
similar example.
