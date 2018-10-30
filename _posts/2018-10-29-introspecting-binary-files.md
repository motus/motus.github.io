---
layout: default
title: Introspecting Binary Files in Linux
---

# Analyzing Linux object files

Object files in Linux come in four different flavors:

* `.o` object file. This is the most basic object file format in Unix. There is usually a 1:1
  correspondence between a source file, e.g. in C, C++, assembly, or Fortran.
* `.a` static library. A static library in Linux is just a collection of `.o` object files archived
  into one file. That is, `.o` file to `.a` file is like `.class` file to `.jar` file in Java.
* `.so` dynamic library.
* An executable (usually in ELF format).

In this tutorial you will learn how to introspect all four types of binary files.

<!-- MORE -->

## Dynamic dependencies

Oftentimes we need to see what *other* dynamic libraries a given `.so` file depends on. This is
where `ldd` utility comes in handy:

```text
$ ldd libcaffe2.so
        linux-vdso.so.1 =>  (0x00007fffc006a000)
        libc10.so => /home/sergiym/devel/libtorch/lib/libc10.so (0x00007f5381190000)
        librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f5380f80000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f5380d60000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f5380b40000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f5380920000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f5380610000)
        libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f5380270000)
        libgomp-7bcb08ae.so.1 => /home/sergiym/devel/libtorch/lib/libgomp-7bcb08ae.so.1 (0x00007f5380040000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f537fc70000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f5389600000)
```

`ldd` can be applied to both `.so` and executable files. This tool is very helpful to make sure a
certain library or an application is portable, or see what dynamic dependencies are missing.

## Static libraries

Static libraries in Unix are just uncompressed archives of `.o` files. It is possible to extract an
individual object file out of archive, or to add a new file to it. For that, every Unix system as an
`ar` command ("ar" stands for "archive"; it is very similar in usage to its `tar` "tape archive"
cousin).

To list `.o` files from a static library, run `ar tv` ("table/verbose"), e.g.:

```text
$ ar tv libonnx.a
rw-r--r-- 0/0  84328 Oct 22 00:20 2018 defs.cc.o
rw-r--r-- 0/0  37888 Oct 22 00:20 2018 schema.cc.o
rw-r--r-- 0/0 152472 Oct 22 00:20 2018 checker.cc.o
rw-r--r-- 0/0   9464 Oct 22 00:19 2018 assertions.cc.o
rw-r--r-- 0/0  96096 Oct 22 00:20 2018 interned_strings.cc.o
rw-r--r-- 0/0 263568 Oct 22 00:20 2018 ir_pb_converter.cc.o
rw-r--r-- 0/0   9344 Oct 22 00:20 2018 model_helpers.cc.o
rw-r--r-- 0/0  10936 Oct 22 00:19 2018 status.cc.o
rw-r--r-- 0/0  92480 Oct 22 00:20 2018 defs.cc.o
rw-r--r-- 0/0  54864 Oct 22 00:20 2018 data_type_utils.cc.o
rw-r--r-- 0/0 184264 Oct 22 00:20 2018 defs.cc.o
. . . . .
```

## Listing symbols

### nm

There are several ways to list the symbol names available in such file. The simplest one is `nm`
command ("nm" stands for "name"). It works for all four types of object files, including executables
(if their symbol table wasn't stripped). For static libraries, `nm` will also print the names of
`.o` files that contain each symbol.

A useful incantation is `nm -gC --defined-only`, e.g.

```text
$ nm -gC --defined-only libcaffe2.so
0000000007f5b1e0 D backend_lib
0000000007f609c0 B __bss_start
00000000014251b0 T cblas_dcopy
0000000001422590 T cblas_dgemm_batch
0000000001422560 T cblas_dscal

. . . . .

0000000007e28780 V vtable for std::_Sp_counted_ptr_inplace<std::string, std::allocator<std::string>, (__gnu_cxx::_Lock_policy)2>
0000000007faf6e0 u onnx_torch::RegisterOneFunctionBuilder(onnx_torch::FunctionBuilder&&)::function_builder_0_status
0000000007faf430 u onnx_torch::OpSchema::all_tensor_types()::all_tensor_types
0000000007faf790 u onnx_torch::OpSchema::all_numeric_types()::all_numeric_types
0000000007faf950 u onnx_torch::OpSchema::numeric_types_for_math_reduction()::numeric_types_for_math_reduction
0000000007f60a88 u at::LegacyTypeDispatch::initForDeviceType(at::DeviceType)::cpu_once
0000000007f60a84 u at::LegacyTypeDispatch::initForDeviceType(at::DeviceType)::cuda_once
0000000007f60a80 u at::LegacyTypeDispatch::initForScalarType(at::ScalarType)::once
00000000007c6710 W at::Context::registerType(at::Backend, at::ScalarType, at::Type*)::{lambda(at::Type*)#1}::_FUN(at::Type*)
```

Here `-C` stands for demangling C++ identifiers, and `-g --defined-only` to print only exported
symbols.

`nm` output are triplets of: symbol value (i.e. address), symbol type, and symbol name. Value is
missing for undefined (i.e. external) symbols. `man nm` explains all symbol types in detail.

### objdump

Another useful tool to inspect the symbol table is `objdump`. Comparing to `nm`, it can print much
more useful information about each symbol and the library. Use `-T` to list dynamic symbols, and
`-C` to demangle C++ names:

```text
$ objdump -TC libcaffe2.so | head

libcaffe2.so:     file format elf64-x86-64

DYNAMIC SYMBOL TABLE:
0000000000443f10 l    d  .init  0000000000000000              .init
0000000007dd6610 l    d  .tdata 0000000000000000              .tdata
0000000000000000      DF *UND*  0000000000000000  GLIBCXX_3.4 std::string::compare(unsigned long, unsigned long, std::string const&) const
0000000000000000      DF *UND*  0000000000000000  GLIBC_2.2.5 dlerror
0000000000000000      DF *UND*  0000000000000000  GLIBCXX_3.4 std::string::replace(unsigned long, unsigned long, char const*, unsigned long)
0000000000000000      DF *UND*  0000000000000000  GLIBC_2.2.5 strdup
```

### readelf

If shared library or executable is in ELF format (which is the case in Linux), `readelf` command can
be useful, too. `-Ws` options stand for printing symbols in wide format.

```text
$ readelf -Ws libcaffe2.so | head

Symbol table '.dynsym' contains 32333 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000443f10     0 SECTION LOCAL  DEFAULT    6
     2: 0000000007dd6610     0 SECTION LOCAL  DEFAULT   14
     3: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _ZNKSs7compareEmmRKSs@GLIBCXX_3.4 (2)
     4: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND dlerror@GLIBC_2.2.5 (3)
     5: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSs7replaceEmmPKcm@GLIBCXX_3.4 (2)
     6: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND strdup@GLIBC_2.2.5 (4)
```

I don't know if `readelf` has options to demangle C++ names or filter symbols by type.
