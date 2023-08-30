# DkSDK FFI

DkSDK is a set of tools to help you build native applications for the
desktop, web, mobile and high-level embedded devices. One of those
tools is **DkSDK FFI**, which is the documentation you are reading
right now.

> If you were looking for another DkSDK tool, there is:
> * DkSDK CMake, at <https://diskuv.com/cmake/help/latest/>, is a tool to manage building C and OCaml source code.

**Problem**: Let's say you want to write code in an obscure, [rarely breaks top 50 popularity](https://spectrum.ieee.org/top-programming-languages-2022) programming language like [OCaml](https://v3.ocaml.org/). Your compiled code must run in diverse runtime environments: Android, iOS, desktop, etc. But you are loathe to throw away the numerous libraries and the vast ecosystem of tools that are in the dominant Java, Swift, and C++ (etc.) environments.

DkSDK FFI is a framework that solves the above problem. One framework component, DkSDK FFI OCaml, is available now, and other framework components like DkSDK FFI Java will be released in the future. Let's look at some example code written in OCaml, but bear in mind two things:

1. Each of the two examples below can be written in *any* DkSDK FFI language.
2. We'll explain what is going on after the examples.

Here is an example of registering a DkSDK FFI "COM object" in OCaml.

> When DkSDK FFI Java is released, this example could be written in Java. When DkSDK FFI Swift is released, this example could be written in Swift. And so on.

```ocaml
#use "topfind";;
#require "DkSDKFFIOCaml_Std";;

open DkSDKFFIOCaml_Std ;;

open ComStandardSchema.Make (ComMessage.C)
open Com.MakeClassBuilder (ComMessage.C) ;;

let com = Com.create_c () ;;

let create_object return args =
    let number = Reader.Si32.(i1_get (of_message args)) in
    return (New
        (Printf.sprintf
        "instance constructed with create_object(args = %ld)"
        number)) ;;

let ask ~self return args =
    let question = Reader.St.(i1_get (of_message args)) in
    let ret =
      Printf.sprintf "I am an %s and I was asked: %s" self question
    in
    let bldr = Builder.St.(let r = init_root () in i1_set r ret; r) in
    return (Capnp bldr) ;;

register com ~classname:"Basic::Question::Taker"
      [
        class_method ~name:"create_object" ~f:create_object ();
        instance_method ~name:"ask" ~f:ask ();
      ] ;;
```

Here is an example of using that registered DkSDK FFI COM object from OCaml.

> When DkSDK FFI Java is released, this example could be written in Java. And so on.

```ocaml
module BasicQuestionTaker = struct
  open ComStandardSchema.Make (ComMessage.C)
  let create com =
      Com.borrow_class_until_finalized com "Basic::Question::Taker"
  let method_create_object = Com.method_id "create_object"
  let method_ask = Com.method_id "ask"

  class questiontaker _clazz inst =
    object
      method ask question =
        let args =
          let open Builder.St in
          let rw = init_root () in i1_set rw question; to_message rw
        in
        let ret_ptr =
            Com.call_instance_method inst method_ask args
        in
        Reader.St.i1_get (Reader.of_pointer ret_ptr)
    end

  let new_questiontaker clazz number =
    let args =
      let open Builder.Si32 in
      let r = init_root () in i1_set_int_exn r number; to_message r
    in
    Com.call_class_constructor
      clazz method_create_object (new questiontaker clazz) args
end ;;

let questiontaker_clazz = BasicQuestionTaker.create com
let questiontaker =
  BasicQuestionTaker.new_questiontaker questiontaker_clazz 37 ;;

let () = print_endline (questiontaker#ask "What am I?") ;;

(* Output:
    I am an instance constructed with create_object(args = 37)
    and I was asked: What am I?
*)
```

**Context**: A long time ago in a galaxy far, far away, Microsoft invented the Component Object Model ("COM") for programming languages like Visual Basic, C and Delphi to interoperate on the same machine. COM was an application binary interface (ABI) where COM "objects" were given C++ style virtual tables; subsequently these COM objects had the functionality of simple C++ objects without needing C++. These COM objects also solved cross-language memory management using manual reference counting. And then Microsoft went crazy trying to expand its reach, and unsuccessfully pushed for a COM enhancement (ActiveX) to be the backbone of the "World Wide Web". But COM survived in several places like DirectX, Apple Core Foundation, and Mozilla XPCOM.

**Context**: There is a neat remote procedure call ("RPC") system called Cap n' Proto RPC. It features [zero overhead for encoding and decoding](https://capnproto.org/), bindings to many programming languages, and [promise pipelining](https://capnproto.org/rpc.html) to solve the distributed, cascading latency problem that inspired GraphQL. Think of Cap n' Proto as a very fast JSON framework combined (optionally) with remote procedure calls.

**A concrete example**: You make an Android application using Java in Android Studio, with the business logic (or data layer or "Model" objects) written in an obscure language like OCaml. And an iOS application using Swift in Xcode, re-using the same business logic you already wrote in OCaml.  And a desktop application using a C++ GUI framework, re-using the same business logic. Wrapping your OCaml objects as COM objects is one way for object-oriented languages like Java, Swift and C++ to communicate with your OCaml code. But that sounds like a lot of work, and it could be! And we haven't even mentioned the overhead of calling methods using a C/C++ calling convention from a foreign programming language.

**Enter the DkSDK FFI framework**: DkSDK FFI combines both COM and Cap n' Proto for *same-process* interoperability. We get rid of COM's C/C++ calling convention. That means there is no stack-allocating each function parameter and encoding/decoding each foreign language type as a C native type. Instead each function call has a single argument ... a Cap n' Proto "Struct" (think of it as a JSON object with little or no encoding overhead) ... and a single "Struct" return value. Like COM, DkSDK FFI objects are reference counted. Unlike COM, DkSDK FFI arguments and return values are also reference counted. In other words, DkSDK FFI core concepts are simple, uniform, accessible in many programming languages, and have been shown to work over decades.

## Status

For news about DkSDK FFI,
[![Twitter URL](https://img.shields.io/twitter/url/https/twitter.com/diskuv.svg?style=social&label=Follow%20%40diskuv)](https://twitter.com/diskuv) on Twitter.

Currently DkSDK FFI is in preview for OCaml-ers. Even though DkSDK FFI can be used, for example, to interoperate between Java and Swift, the OCaml language has slight privileges over other languages.
- First is in licensing. When DkSDK FFI Java is released in a few months to the predominant "Maven" Java binary package manager, the binding source code and Java `.so` and `.dll` shared libraries will be available under an ["OSL-3.0" OSI-approved](https://opensource.org/license/osl-3-0-php/) open source license, but only for one OS architecture (probably Linux x86_64 since easy to run with Docker). If you need other OS architectures you can purchase the full source code with a [DkSDK license](https://diskuv.com/pricing), or get it free on request if you are a security engineer, educator or a related-field academic researcher. And when DkSDK FFI Swift is released next year, it will be OSL-3.0 licensed only for one OS architecture (probably Darwin arm64). *DkSDK FFI OCaml?* It will be OSL-3.0 licensed so it can be used alongside *all* other released OSL-3.0 architectures (ex. Linux x86_64, Darwin arm64 and so on).
- Second, there may be some leaky abstractions in DkSDK FFI that help OCaml. One example is DkSDK FFI method identifiers are 31-bit identifiers.

You'll need to wait for your foreign programming language (Java, etc.) to be implemented in DkSDK FFI. Reference material and timelines are:
- DkSDK FFI OCaml - https://diskuv.gitlab.io/distributions/1.0/dksdk-ffi-ocaml/DkSDKFFIOCaml_Std/DkSDKFFIOCaml_Std/index.html. If you browse that `odoc` documentation, be sure to click on its links to modules. There is a lot of documentation in there, but we haven't yet organized it narratively.
- DkSDK FFI C - https://diskuv.com/ffi-c/help/latest/. This is the already-completed kernel of the DkSDK FFI platform. Other language implementations like DkSDK FFI OCaml use it. However, we don't think it makes sense for an end-user to use it directly: writing COM objects in a language without objects like C is verbose, and the [ocaml-ctypes](https://github.com/yallop/ocaml-ctypes) library is mature and relatively easy to use. But the FFI C documentation will help you debug stack traces and generally understand what is going on. And if you want it, full source is available today with a [DkSDK license](https://diskuv.com/pricing)
- DkSDK FFI C++, Java, Swift - When these are ready for public release We'll send announcements. Earliest will be Java in a few months.
- DkSDK FFI Rust, Web (WASM), Python - Since the kernel DkSDK FFI C is C11 standards compliant and is a glorified manipulator of memory buffers, DkSDK FFI should have a straightforward WASM implementation.  But we're not sure if these will get done because we don't need these. If a few DkSDK subscribers need them, We'll prioritize them.
