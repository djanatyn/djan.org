+++
title = "exploring xtensa in llvm"
description = "getting sidetracked running zig on esp32"
date = 2026-11-12
draft = true

[taxonomies]
categories = ["lab notebook"]

[extra]
toc = false
code_copy = true
comment = false
og_image = "/img/todo.png" 
+++

i'm not a confident C++ programmer, and i would like to write as little C++ as possible. i've been largely successful at this, but i sometimes struggle to avoid writing C++ in embedded environments, especially when writing code for the ESP32.

i've been using the excellent [Inkplate Arduino Library](https://github.com/SolderedElectronics/Inkplate-Arduino-library) with [`arduino-cli`](https://docs.arduino.cc/arduino-cli/) for several projects recently, and i was curious if [zig](https://ziglang.org/) could help!

C++ supports C ABI interop by setting [external language linkage](https://en.cppreference.com/cpp/language/language_linkage) to "C", and
zig has [excellent support](https://ziglang.org/documentation/master/#C) for interacting with the C ABI:

{% <quote cite="[Why Zig When There is Already C++, D, and Rust?](https://ziglang.org/intro/why_zig_rust_d_cpp/)"> %}
Currently it is pragmatically true that C is the most versatile and portable language. Any language that does not have the ability to interact with C code risks obscurity. Zig is attempting to become the new portable language for libraries by simultaneously making it straightforward to conform to the C ABI for external functions, and introducing safety and language design that prevents common bugs within the implementations.
{% </quote> %}

{% <quote cite="[Export functions, variables, and types for C code to depend on](https://ziglang.org/learn/overview/)"> %}
One of the primary use cases for Zig is exporting a library with the C ABI for other programming languages to call into. The export keyword in front of functions, variables, and types causes them to be part of the library API:
```zig
export fn add(a: i32, b: i32) i32 {
    return a + b;
}
```
{% </quote> %}

i would love to spend more time writing zig, and i would strongly prefer to use zig instead of C++ for my inkplate experiments.

my usage of the inkplate API is very simple. while i [may not be able to call it directly](https://en.wikipedia.org/wiki/Name_mangling) from zig, i could create a thin wrapper providing a C-ABI interface to the library functions i need.

can zig save the day?

# step 1: emit xtensa

my hardware target is the [inkplate tempera4](https://soldered.com/products/inkplate-4-tempera), which uses an ESP32 microcontroller. this microcontroller uses the Xtensa 

---

# `fixup value must be 4-byte aligned`

i kept tinkering with `build.zig` and eventually got to a state where i was *almost* always hitting this compiler error:
```bash
~/code/zig-inkplate: ~/code/zig/build/stage3/bin/zig build firmware
firmware
└─ run arduino-cli
   └─ compile obj zig_app debug xtensa-freestanding-none failure
error: error: <unknown>:0: fixup value must be 4-byte aligned


error: process exited with code 1
failed command: /Users/jstrickland/code/zig/build/stage3/bin/zig build-obj -target xtensa-freestanding-none -mcpu esp32 -Mroot=zig/demo.zig --cache-dir .zig-cache --global-cache-dir /Users/jstrickland/.cache/zig --build-root . --name zig_app --zig-lib-dir /Users/jstrickland/code/zig/build/stage3/lib/zig/ --listen=-

Build Summary: 0/3 steps succeeded (1 failed)
firmware transitive failure
└─ run arduino-cli transitive failure
   └─ compile obj zig_app debug xtensa-freestanding-none failure
```

i was getting strange and intermittent behavior. i tried to isolate to the smallest reproducible case i could find, but the error was inconsistent. sometimes i could compile calling inkplate API functions from my shim with no issues:

```bash
cat zig/demo.zig && ~/code/zig/build/stage3/bin/zig build firmware
```
```zig
const shim = @import("shim");

export fn zig_app_setup() void {
    shim.serial_print("hello world");
}

export fn zig_app_loop() void {
    shim.serial_print("hello world");
}
```
```
/Users/jstrickland/Library/Arduino15/packages/soldered-inkplate-boards/tools/esp-x32/2511/bin/../lib/gcc/xtensa-esp-elf/14.2.0/../../../../xtensa-esp-elf/bin/ld: warning: _floatdidf.o: missing .note.GNU-stack section implies executable stack
/Users/jstrickland/Library/Arduino15/packages/soldered-inkplate-boards/tools/esp-x32/2511/bin/../lib/gcc/xtensa-esp-elf/14.2.0/../../../../xtensa-esp-elf/bin/ld: NOTE: This behaviour is deprecated and will be removed in a future version of the linker
Sketch uses 957503 bytes (30%) of program storage space. Maximum is 3145728 bytes.
Global variables use 90156 bytes (27%) of dynamic memory, leaving 237524 bytes for local variables. Maximum is 327680 bytes.
```

i tried experimenting with different `callconv` expressions, using `addTranslateC` in build.zig, but nothing worked seemed to work consistently.
```bash
cat zig/demo.zig && ~/code/zig/build/stage3/bin/zig build-obj zig/demo.zig \
    -target xtensa-freestanding-none -mcpu=esp32 \
    -O ReleaseSmall -femit-bin=/tmp/demo-extern-decl.o
echo "exit=$?"
```
```zig
extern fn serial_print(text: [*:0]const u8) callconv(.c) void;

export fn zig_app_setup() void {
    serial_print("starting!");
}

export fn zig_app_loop() void {}
```
```
error: <unknown>:0: fixup value must be 4-byte aligned

exit=1
```

at the time, i worked through the issue with some LLM-assistance: after explaining my predicament, an open weight model was able to isolate the issue in the LLVM source code, identify the problem, and to apply a patch which fixed my compilation issues without little prompting.

since then, i've spent some time studying:
- why did this error occur?
- what was the code causing even trying to do?
- why does the fix solve the problem?

i learned about the Xtensa ISA, the `l32r` instruction, and LLVM compilers internals like `MCFixup`. if you're interested in that kind of thing, keep reading!

---

# the llvm experimental xtensa backend

to translate your LLVM IR into machine code for a particular target, you'll need a backend for that architecture.

{% <quote cite="[[RFC] Tensilica Xtensa (ESP32 Backend)](https://discourse.llvm.org/t/rfc-tensilica-xtensa-esp32-backend/57835)"> %}
Hi All!

On behalf of Espressif Systems, I would like to propose Xtensa backend to be added as an experimental to LLVM project.
{% </quote> %}

# viewing the assembly for our zig code

we'll use this file as our example:
```zig
~/code/zig-inkplate: cat test.zig
extern fn serial_print(text: [*:0]const u8) callconv(.c) void;

export fn zig_app_setup() void {
    serial_print("starting!");
}

export fn zig_app_loop() void {}
```

this code links to an external function `serial_print` and calls it with a string. it's ok that we didn't provide a definition for `serial_print` - this is the shape we expect for our zig code, as C++ will provide the implementation for these symbols when we build our firmware image.

we can run `zig build-obj` with the `-emit-asm` option to see the assembly for `test.zig_app_setup`:
```asm
test.zig_app_setup:
.Lfunc_begin1:
        .cfi_startproc
        .loc    6 4 17 prologue_end
        entry   a1, 32
        or      a7, a1, a1
        .cfi_def_cfa a7, 32
        l32r    a10, .LCPI1_0
        l32r    a8, .LCPI1_1
        callx8  a8
        retw.n
```

# what is a fixup?

["fixups"](https://www.embecosm.com/appnotes/ean10/html/ch06s02.html)  represent *currently-unknown instruction data* during instruction encoding. often, the assembler knows that it needs to place instruction, but the data to which the instruction refers to isn't available. this can occur with external symbols, such as `serial_print` (linkage), or with data literals that may be present in other sections of the file (placement).

if these fixup values do eventually become known, the assembler can replace them with concrete memory locations during [relaxation](https://eli.thegreenplace.net/2013/01/03/assembler-relaxation). values that cannot be resolved are converted to [relocations](https://en.wikipedia.org/wiki/Relocation_(computing)), which the linker will handle when appropriate symbols become resolvable.

"fixups" are an assembler internal, while relocations exist concretely as bytes inside binaries. [every relocation was once a fixup!](https://github.com/lowRISC/riscv-llvm/blob/master/docs/06-relocations-and-fixups.mkd) relocations are visible inside object files with tools like `objdump` or `readelf`, and usually have named sections (such as `SHT_RELA` and `SHT_REL` in ELF).

the fixup type we're dealing with in this error is an `MCFixup` instance:
{% <quote cite="[`llvm/include/llvm/MC/MCFixup.h`](https://github.com/llvm/llvm-project/blob/ca7933e47d3a3451d81e72ac174dcb5aa28b59d1/llvm/include/llvm/MC/MCFixup.h#L47-L81)"> %}
```
/// Encode information on a single operation to perform on a byte
/// sequence (e.g., an encoded instruction) which requires assemble- or run-
/// time patching.
///
/// Fixups are used any time the target instruction encoder needs to represent
/// some value in an instruction which is not yet concrete. The encoder will
/// encode the instruction assuming the value is 0, and emit a fixup which
/// communicates to the assembler backend how it should rewrite the encoded
/// value.
///
/// During the process of relaxation, the assembler will apply fixups as
/// symbolic values become concrete. When relaxation is complete, any remaining
/// fixups become relocations in the object file (or errors, if the fixup cannot
/// be encoded on the target).
```
{% </quote> %}

while it wasn't clear i was expecting to see some relocations in my object file.

`build.zig` was configured to create an object file `zig_app` using `wrapper.h` (not `wrapper.cpp`!), so the implementation for the external functions `demo.zig` called was not available to LLVM or the zig compiler:
```zig
    const inkplate = b.addTranslateC(.{
        .root_source_file = b.path("arduino/wrapper.h"),
        .target = target,
        .optimize = optimize,
        .link_libc = false,
    });

    const app_module = b.createModule(.{
        .root_source_file = b.path("zig/demo.zig"),
        .target = target,
        .optimize = optimize,
        .imports = &.{
            .{
                .name = "inkplate",
                .module = inkplate.createModule(),

            },
        },
    });

    const app = b.addObject(.{
        .name = "zig_app",
        .root_module = app_module,
    });
```

this means i expect the object file `zig_app.o` to *have relocations for the external functions i call from `wrapper.h`* when compiled successfully, which the linker will eventually handle.

# `XtensaAsmBackend::adjustFixupValue`

with that context provided, i can reveal that our error is emitted from the `XtensaAsmBackend` `adjustFixupValue` function!
{% <quote cite="[`llvm/lib/Target/Xtensa/MCTargetDesc/XtensaAsmBackend.cpp`](https://github.com/llvm/llvm-project/blob/ca7933e47d3a3451d81e72ac174dcb5aa28b59d1/llvm/lib/Target/Xtensa/MCTargetDesc/XtensaAsmBackend.cpp#L117-L125)"> %}
```c++
case Xtensa::fixup_xtensa_l32r_16:
  unsigned Offset = Fixup.getOffset();
  if (Offset & 0x3)
    Value -= 4;
  if (!isInt<18>(Value) && (Value & 0x20000))
    Ctx.reportError(Fixup.getLoc(), "fixup value out of range");
  if (Value & 0x3)
    Ctx.reportError(Fixup.getLoc(), "fixup value must be 4-byte aligned");
  return (Value & 0x3fffc) >> 2;
```
{% </quote> %}

`Xtensa::fixup_xtensa_l32r_16` is a particular `Kind` of `MCFixup` value, for an `xtensa` instruction, `l32r` in particular. `l32r` is short for "Load 32-bit PC-Relative":

> L32R is a PC-relative 32-bit load from memory. It is typically used to load constant values into a register when the constant cannot be encoded in a MOVI instruction.

it loads with an offset relative to the current program counter, so it can specify any address up to 256KB *before* its location:

{% <quote cite="Xtensa Instruction Set Architecture (ISA) Reference Manual"> %}
L32R forms a virtual address by adding the 16-bit one-extended constant value encoded in the instruction word shifted left by two to the address of the L32R plus three with the two least significant bits cleared. Therefore, **the offset can always specify 32-bit aligned addresses from -262141 to -4 bytes from the address of the L32R instruction.** 32 bits (four bytes) are read from the physical address.
{% </quote> %}

importantly, the stored constant is shifted left by two at execution time, which effectively multiplies it by 4 (`2^2=4`). this means the encoder must *divide* the byte distance by 4 to fit it into the field. if the byte distance of the data from the instruction cannot be divided by 4, it can't be represented, which is what the fixup error means: "value must be 4-byte aligned".

this makes sense given what we know about `l32r`, as the instruction's word can only be determined once we've placed the data literal it refers to.

what emits those literals? `XtensaAsmPrinter::emitConstantPool()`! 
{% <quote cite=""> %}
```c++
// EmitConstantPool - Print to the current output stream assembly
// representations of the constants in the constant pool MCP. This is
// used to print out constants which have been "spilled to memory" by
// the code generator.
void XtensaAsmPrinter::emitConstantPool() {
  const Function &F = MF->getFunction();
  const MachineConstantPool *MCP = MF->getConstantPool();
  const std::vector<MachineConstantPoolEntry> &CP = MCP->getConstants();
  if (CP.empty())
    return;

  OutStreamer->pushSection();

  auto *TS =
      static_cast<XtensaTargetStreamer *>(OutStreamer->getTargetStreamer());
  MCSection *CS = getObjFileLowering().SectionForGlobal(&F, TM);
  TS->startLiteralSection(CS);

  int CPIdx = 0;
  for (const MachineConstantPoolEntry &CPE : CP) {
    emitMachineConstantPoolEntry(CPE, CPIdx++);
  }

  OutStreamer->popSection();
}
```
{% </quote> %}

// TODO: ?
these constants go in their own section, which is created by `XtensaTargetElfStreamer::startLiteralSection`:
{% <quote cite="[`llvm/lib/Target/Xtensa/MCTargetDesc/XtensaTargetStreamer.cpp`](https://github.com/llvm/llvm-project/blob/ca7933e47d3a3451d81e72ac174dcb5aa28b59d1/llvm/lib/Target/Xtensa/MCTargetDesc/XtensaTargetStreamer.cpp#L106-L115)"> %}
```c++
void XtensaTargetELFStreamer::startLiteralSection(MCSection *BaseSection) {
  MCContext &Context = getStreamer().getContext();

  std::string SectionName = getLiteralSectionName(BaseSection->getName());

  MCSection *ConstSection = Context.getELFSection(
      SectionName, ELF::SHT_PROGBITS, ELF::SHF_EXECINSTR | ELF::SHF_ALLOC);

  ConstSection->setAlignment(Align(4));
}
```
{% </quote> %}

we can see that it's setting the alignment to 4 already! it looks like it's operation on a `ConstSection` with a name retrieved by `getLiteralSectionName`. let's look at that function as well: 

<!-- ~/code/llvm-project-22/llvm/lib/Target/Xtensa/MCTargetDesc/XtensaTargetStreamer.cpp -->
{% <quote cite="[TODO](https://github.com/llvm/llvm-project/blob/ca7933e47d3a3451d81e72ac174dcb5aa28b59d1/llvm/lib/Target/Xtensa/MCTargetDesc/XtensaTargetStreamer.cpp#L106-L115)"> %}
```c++
static std::string getLiteralSectionName(StringRef CSectionName) {
  std::size_t Pos = CSectionName.find(".text");
  std::string SectionName;
  if (Pos != std::string::npos) {
    SectionName = CSectionName.substr(0, Pos);

    if (Pos > 0)
      SectionName += ".text";

    CSectionName = CSectionName.drop_front(Pos);
    CSectionName.consume_front(".text");

    SectionName += ".literal";
    SectionName += CSectionName;
  } else {
    SectionName = CSectionName;
    SectionName += ".literal";
  }
  return SectionName;
}
```
{% </quote> %}

---

sources :
- https://stackoverflow.com/questions/19532826/what-does-a-dangerous-relocation-error-mean
