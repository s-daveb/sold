# Mach-O linker overview

Mach-O is an executable and linkable file format. It was originally
created for the Mach kernel-based system. Since NeXTSTEP based on the Mach
kernel and the modern macOS/iOS is a direct descendent of it, Mach-O is
now universally used as an object file on Apple's systems.

In this document, I'll explain Mach-O by comparing it with ELF.

## Sections and subsections

Section is not an atomic unit of copying in Mach-O. If a section has the
MH_SUBSECTIONS_VIA_SYMBOLS flag, it can be split at locations where
symbols in the section points to. Splitted section is called "subsection".

For example, assume your source file contains two function definitions,
`foo` and `bar`. The compiled object file for the source file contains
only one text section which consists of two subsections. One subsection
starts at where `foo` refers to, and the other starts at where `bar`
refers to.

So, Mach-O's section is not equivalent to ELF's section; Mach-O's
subsection is.

# Relocations

A Mach-O relocation can refer to either a symbol or a section, while an
ELF relocation can refer to only a symbol.

# Dynamic name resolution

In Mach-O, dynamic symbols are resolved with a tuple of (dylib's name,
symbol name). That means dynamic symbols must be resolved by their
corresponding dynamic libraries at runtime.

This is contrary to ELF in which dynamic symbols are resolved only by
name. In ELF, dynamic symbols can be resolved by any dynamic library as
long as their names match.

Mach-O's dynamic symbols are stored as a trie so that symbols' common
prefixes are shared in a file. In ELF, dynamic symbol strings are just a
run of NUL-terminated strings. So, Mach-O is more complicated but compact.

# Lazy function symbol resolution

Lazy function symbol resolution is done in Mach-O in the same way as ELF
but with slightly different file layout.

`__stubs` section contains PLT entries. It reads a function address from
`__la_symbol_ptr` section and jump there.

`__la_symbol_ptr` entries are initialized to point to entries in
`__stubs_helper` section. If a PLT entry is called for the first time, the
control is transferred to its corresponding entry in `__la_symbol_ptr`. It
then calls dyld_stub_binder with appropriate arguments for symbol
resolution.

There's no notion of "canonical PLT" in Mach-O because the compiler always
emit code to load a function pointer value from GOT even for `-fno-PIC`.
In other words, the compiler always assumes that the address of a function
is not known at link-time. Therefore, we'll never see an object file which
assumes that a function address is a link-time constant.

# Re-exported libraries

Mach-O dylibs can refer other dylibs as "reexported libraries". If
`libfoo.dylib` refers `libbar.dylib` as a reexported library, all symbols
defined by `libbar.dylib` become available as if they were defined by
`libfoo.dylib`.

The purpose of reexporting is to give developers freedom to reorganize their
libraries while keeping the programming interface the same. For example,
macOS's `libSystem.dylib` provides basic functionalities for macOS apps. Apple
developers may want to move some features out of `libSystem.dylib` to a new
dylib. They can safely do this by referring the new dylib as a reexported
library of `libSystem.dylib`. If there's no reexporting feature, they can't do
this without asking other developers to add a new `-l` line to their build
file.

# Thread-local variables

If a symbol is of thread-local variable (TLV), it is defined in a
`__thread_vars` section in an input file. A `__thread_vars` consists of
a vector of three-word tuples of

1. a function pointer to `tlv_get_addr`,
2. an indirect pointer to the TLS block of the Mach-O image, and
3. the offset within the TLS block.

. You can obtain the address of a TLV by calling the function pointer
with the address of the tuple as an argument. `tlv_get_addr` then adds
the TLS block address and the offset to compute the address of the TLV.

TLVs are always referenced by `ARM64_RELOC_TLVP_LOAD_*` relocations (or
`X86_64_RELOC_TLV` relocation on x86-64). For each TLV referenced by these
relocations, the linker creates an entry in the linker-synthesized
`__thread_ptrs` section and let it refer to the address of its corresponding
`tlv_get_addr` entry.

TLVs are always referenced indirectly via `__thread_ptrs`. If a TLV is
defined locally or resolved to the same Mach-O file, its `__thread_ptrs`
entry refers to its `__thread_vars` entry in the same file. Otehrwise,
the `__thread_ptr` entry refers to a `__thread_vars` entry in other Mach-O
file image.

# Endianness and bitwidth

Since Mach-O is effectively used only by Apple, and all Apple systems are
little-endian, Mach-O is effectively little-endian only. Likewise, it is
64-bit only since Apple has deprecated 32-bit apps.

Apple Watch seems to be using an ILP32 ABI on ARM64, but I don't know the
details about it.