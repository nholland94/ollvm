# ollvm-tapir

This fork of ollvm adds support for the extra instructions added by [LLVM-Tapir](https://github.com/wsmoses/Parallel-IR). It adds the `detach`, `reattach`, and `sync` instructions to ollvm.

Here is an example program which generates a llvm module which implements the fibonacci sequence using the additional LLVM-Tapir instructions.

OCaml Program:

```ocaml
open Ollvm.Ez.Value
open Ollvm.Ez.Instr
open Ollvm.Ez.Block
module M = Ollvm.Ez.Module
module T = Ollvm.Ez.Type
module P = Ollvm.Printer

let _ =
  let m = M.init
    "tapir_fib_test"
    ("x86_64", "pc", "linux-gnu")
    "e-m:e-i64:64-f80:128-n8:16:32:64-S128"
  in
  let (m, cond) = M.local m T.i1 "cond" in
  let (m, x) = M.local m (T.pointer T.i32) "x" in
  let (m, [x0; x1; x2; y; n; r; n_sub_1; n_sub_2]) =
    M.locals m T.i32 ["x0"; "x1"; "x2"; "y"; "n"; "r"; "n_sub_1"; "n_sub_2"]
  in
  let (m, [l_entry; l_else; l_det; l_cont; l_merge; l_exit]) =
    M.locals m T.label ["entry"; "else"; "det"; "cont"; "merge"; "exit"]
  in
  let (m, fib) = M.global m T.i32 "fib" in
  let fib_f = define fib [n] [
    block l_entry [
      x <-- alloca T.i32;
      cond <-- ult n (i32 2);
      br cond l_exit l_else ];
    block l_else [
      detach l_det l_cont ];
    block l_det [
      n_sub_1 <-- sub n (i32 1);
      x0 <-- call fib [n_sub_1];
      let (_, i) = store x0 x in i;
      reattach l_cont ];
    block l_cont [
      n_sub_2 <-- sub n (i32 2);
      y <-- call fib [n_sub_2];
      sync l_merge ];
    block l_merge [
      x1 <-- load x;
      x2 <-- add x1 y;
      br1 l_exit ];
    block l_exit [
      r <-- phi [(n, l_entry); (x2, l_merge)];
      ret r ] ]
  in
  let m = M.definition m fib_f in
  P.modul (P.empty_env ()) Format.std_formatter m.m_module
```

Output:

```
; ModuleID = 'tapir_fib_test'
target triple = "x86_64-pc-linux-gnu"
target datalayout = "e-m:e-i64:64-f80:128-n8:16:32:64-S128"


define i32 @fib(i32 %n) {
entry:
        %x = alloca i32
        %cond = icmp ult i32 %n, 2
        br i1 %cond, label %exit, label %else
else:
       detach label %det, label %cont
det:
      %n_sub_1 = sub i32 %n, 1
      %x0 = call i32 @fib(i32 %n_sub_1)
      reattach label %cont
cont:
       %n_sub_2 = sub i32 %n, 2
       %y = call i32 @fib(i32 %n_sub_2)
       sync label %merge
merge:
        %x1 = load i32* %x
        %x2 = add i32 %x1, %y
        br label %exit
exit:
       %r = phi i32 [%n, %entry], [%x2, %merge]
       ret i32 %r
}
```

The original README.md for ollvm is below:

# ollvm 0.99

ollvm library offers an interface to manipulate LLVM IR in pure OCaml.
LLVM already provides binding to its C API, but it still mainly imperative
programming with heavy use of side effects.

ollvm is different in the way that you will manipulate OCaml structures
(lists, records, variants, ...). An Ez interface is provided
to make LLVM IR writing pleasant.

You may also want to use LLVM IR, but not the whole LLVM compiler
infrastructure. ollvm allows you to stay independent from llvm library.
You **may** want to bind you code to official bindings with the provided
gateway, but you **do not have to**. Make your optimization passes, use
your own back-end if you please.

### Ez interface utilisation example

Input program:

```ocaml
open Ollvm.Ez.Value
open Ollvm.Ez.Instr
open Ollvm.Ez.Block
module M = Ollvm.Ez.Module
module T = Ollvm.Ez.Type
module P = Ollvm.Printer

let _ =
  (* module initialization *)
  let m = M.init
            "name"
            ("x86_64", "pc", "linux-gnu")
            "e-m:e-i64:64-f80:128-n8:16:32:64-S128" in

  (* variables declaration *)
  let (m, x0) =
    M.local m T.i1 "" in
  let (m, [x1; x2; x3; x4]) =
    M.locals m T.i32 [""; ""; ""; ""] in
  let (m, [entry_b; then_b; else_b]) =
    M.locals m T.label ["entry"; "then"; "else" ] in
  let (m, fact) = M.global m T.i32 "fact" in

  (* fact function definition *)
  let f = define fact [x4]
                 [ block entry_b [
                           x0 <-- eq x4 (i32 0) ;
                           br x0 then_b else_b ; ] ;
                   block then_b [
                           ret (i32 1) ; ] ;
                   block else_b [
                           x1 <-- sub x4 (i32 1) ;
                           x2 <-- call fact [x1] ;
                           x3 <-- mul x4  x2 ;
                           ret x3 ; ] ] in

  (* fact function registration in module *)
  let m = M.definition m f in
  P.modul (P.empty_env ()) Format.std_formatter m.m_module
```

Output:

```
; ModuleID = 'name'
target triple = "x86_64-pc-linux-gnu"
target datalayout = "e-m:e-i64:64-f80:128-n8:16:32:64-S128"

define i32 @fact(i32 %0) {
entry:
        %1 = icmp eq i32 %0, 0
        br i1 %1, label %then, label %else
then:
       ret i32 1
else:
       %2 = sub i32 %0, 1
       %3 = call i32 @fact(i32 %2)
       %4 = mul i32 %0, %3
       ret i32 %4
}
```

### Resources

* [Ollvm on Github](http://www.github.com/OCamlPro/ollvm): latest sources
  in the official GIT repository.
* [Official LLVM website](http://llvm.org/): more details on LLVM and
  its itermediate representation specification.
* [Online documentation](http://sagotch.github.io/ollvm)

### Install

* Via opam: ` opam install ollvm `
* From source: ` ./configure && make && [sudo] make install `

### Limitations

Ollvm is still work in progress.

* Parser misses some LLVM IR features (e.g. metadata attached to
  instructions).
* Ollvm does not checks types of your instruction, nor it ensures
  SSA form (i.e. you can write wrong LLVM IR).
  It *may* ensure good typing (statically) of your LLVM code in futur.

### Known bugs

* `Ez` / `Printer` modules:
  * Unnamed labels are not correctly printed with.
  * Unnamed function arguments are printed in prototypes while they
     should not.

Note that passing though Llvmgateway and printing with official
binding results in a correct LLVM IR output.
