+++
title = "using the inkplate C++ API on ESP32, from zig"
description = "replacing C++ in my life one step at a time"
date = 2026-11-14
draft = false

[taxonomies]
categories = ["lab notebook"]

[extra]
toc = false
code_copy = true
comment = false
og_image = "/img/zig_inkplate_hello_world.png" 
+++

i'm not a confident C++ programmer, and i would like to write as little C++ as possible.

i've been largely successful at this, but i sometimes struggle in embedded environments, especially when writing code for the ESP32.

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

# identifying platform support

my hardware target is the [inkplate tempera4](https://soldered.com/products/inkplate-4-tempera), which uses an ESP32 microcontroller. this microcontroller uses the [xtensa](https://github.com/espressif/xtensa-isa-doc) isa, so i'll need to set my `build.zig` appropriately.

zig has a list of [support platforms](https://ziglang.org/learn/platform-support/), and [`xtensa-linux`](https://ziglang.org/learn/platform-support/xtensa-linux/) is currently tier 4:

> The compiler can generate assembly or C source code for this target.

that's good enough for me! hovering over the "code gen." column in the support table shows that it requires an LLVM experimental backend.

even though the backend is still experimental, it's been around for a while. it looks like it's under active development by Espressif, with an early proposal in march 2021:

{% <quote cite="[[RFC] Tensilica Xtensa (ESP32 Backend)](https://discourse.llvm.org/t/rfc-tensilica-xtensa-esp32-backend/57835)"> %}
Hi All!

On behalf of Espressif Systems, I would like to propose Xtensa backend to be added as an experimental to LLVM project.
{% </quote> %}

# compiling llvm with the experimental xtensa backend

compiling llvm takes a while, but wasn't too tricky. the `README.md` for zig has a recommended approach that i adapted:
{% <quote cite="[Building LLVM, LLD, and Clang from Source](https://codeberg.org/ziglang/zig#release)"> %}
```bash
cd ~/Downloads
git clone --depth 1 --branch release/22.x https://github.com/llvm/llvm-project llvm-project-22
cd llvm-project-22
git checkout release/22.x

mkdir build-release
cd build-release
cmake ../llvm \
  -DCMAKE_INSTALL_PREFIX=$HOME/local/llvm22-assert \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS="lld;clang" \
  -DLLVM_ENABLE_LIBXML2=OFF \
  -DLLVM_ENABLE_TERMINFO=OFF \
  -DLLVM_ENABLE_LIBEDIT=OFF \
  -DLLVM_ENABLE_ASSERTIONS=ON \
  -DLLVM_ENABLE_ZSTD=OFF \
  -DLLVM_PARALLEL_LINK_JOBS=1 \
  -G Ninja
ninja install
```
{% </quote> %}

i added a [cmake flag](https://llvm.org/docs/CMake.html#llvm-related-variables) to enable xtensa:
```bash
-DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD=Xtensa
```

there's a few different experimental targets for the curious, all defined in [`llvm/CMakeLists.txt`](https://github.com/llvm/llvm-project/blob/ca7933e47d3a3451d81e72ac174dcb5aa28b59d1/llvm/CMakeLists.txt#L541-L547):
```
set(LLVM_ALL_EXPERIMENTAL_TARGETS
  ARC
  CSKY
  DirectX
  M68k
  Xtensa
)
```

the build completed without any issues. after that, i compiled zig from a checkout, again following patterns established in `README.md`:
```bash
git checkout 0.17.1
mkdir build && cd build
cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_PREFIX_PATH=$HOME/local/llvm22-xtensa \
  -DZIG_EXTRA_BUILD_ARGS="-Dllvm-has-xtensa"
cmake --build . --parallel
```

that was all it took to get a zig compiler which could emit xtensa machine code!
```bash
~/code/zig-inkplate: ~/code/zig/build/stage3/bin/zig build-lib \
  --show-builtin \
  -target xtensa-freestanding-none \
  -mcpu=esp32
```
```zig
const std = @import("std");
pub const zig_version = std.SemanticVersion.parse(zig_version_string) catch unreachable;
pub const zig_version_string = "0.17.1";
pub const zig_backend = std.lang.CompilerBackend.stage2_llvm;

pub const output_mode: std.lang.OutputMode = .Lib;
pub const link_mode: std.lang.LinkMode = .static;
pub const unwind_tables: std.lang.UnwindTables = .async;
pub const is_test = false;
pub const single_threaded = false;
/// Deprecated; to be removed in 0.18.0. Use `target.abi` instead.
pub const abi: std.Target.Abi = .none;
/// Deprecated; to be removed in 0.18.0. Use `target.cpu` instead.
pub const cpu: std.Target.Cpu = .{
    .arch = .xtensa,
    .model = &std.Target.xtensa.cpu.esp32,
    .features = std.Target.xtensa.featureSet(&.{
      // ...
    }),
};
```

# using `build.zig` to build object files for xtensa

i was quite mesmerized by the [zig build system](https://ziglang.org/learn/build-system/)!
{% <quote> %}
The Zig Build System provides a cross-platform, dependency-free way to declare the logic required to build a project. With this system, the logic to build a project is written in a build.zig file, using the Zig Build System API to declare and configure build artifacts and other tasks. 
{% </quote> %}

to start, i needed to represent the `arduino/wrapper.h` header file that i'll be referring to from `demo.zig`:
```c
#ifndef INKPLATE_SHIM_H
#define INKPLATE_SHIM_H

#ifdef __cplusplus
extern "C" {
#endif

void display_begin(void);
void display_display(void);
void display_clear(void);
void display_set_cursor(int x, int y);
void display_print(const char *text);
void serial_print(const char *text);

#ifdef __cplusplus
}
#endif

#endif
```

this header file contains the function signatures for the symbols i'll be calling. the implementation of these functions is in `arduino/wrapper.cpp`, which the linker will use to resolve relocations in our zig object file at the final compilation stage.

here's how we specify `wrapper.h`:
```zig
const target = b.resolveTargetQuery(.{
    .cpu_arch = .xtensa,
    .os_tag = .freestanding,
    .abi = .none,
    .cpu_model = .{ .explicit = &std.Target.xtensa.cpu.esp32 },
});

const optimize: std.builtin.Optimize = .Debug;

const inkplate = b.addTranslateC(.{
    .root_source_file = b.path("arduino/wrapper.h"),
    .target = target,
    .optimize = optimize,
    .link_libc = false,
});
```

- we use `resolveTargetQuery` to specify the architecture,
- we set the optimization level to `.Debug`, and
- we use `addTranslateC` to import `arduino/wrapper.h`

we can use this value, `inkplate`, to define a module which *imports* the C header definitions as a translated zig module:
```zig
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
```

and finally, `demo.zig` can import `inkplate` to access the library functions through a structured module interface:
```zig
const inkplate = @import("inkplate");

export fn zig_app_setup() void {
    inkplate.serial_print("starting!");
    inkplate.display_begin();
    inkplate.display_clear();
}

export fn zig_app_loop() void {
    inkplate.display_set_cursor(100, 0);
    inkplate.display_print("hello world");
    inkplate.display_display();
}
```

and once we have our compiled module, we can tell `build.zig` to generate an object file with `addObject`:
```zig
const app = b.addObject(.{
    .name = "zig_app",
    .root_module = app_module,
});
```

that's almost all of the code in the whole project! clearly, we're not doing very much, but the goal is to see what's possible before we get ambitious.

# `fixup value must be 4-byte aligned`

i had a persistent and frustrating issue while i was tinkering with `build.zig`. i was *almost* always hitting this compiler error:
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

the behavior was strange and intermittent. i tried to isolate to the smallest reproducible case i could find, but failure was inconsistent. sometimes i could compile calling inkplate API functions from my shim with no issues:

```bash
cat zig/demo.zig && ~/code/zig/build/stage3/bin/zig build firmware
```
```zig
const inkplate = @import("inkplate");

export fn zig_app_setup() void {
    inkplate.serial_print("hello world");
}

export fn zig_app_loop() void {
    inkplate.serial_print("hello world");
}
```
```
/Users/jstrickland/Library/Arduino15/packages/soldered-inkplate-boards/tools/esp-x32/2511/bin/../lib/gcc/xtensa-esp-elf/14.2.0/../../../../xtensa-esp-elf/bin/ld: warning: _floatdidf.o: missing .note.GNU-stack section implies executable stack
/Users/jstrickland/Library/Arduino15/packages/soldered-inkplate-boards/tools/esp-x32/2511/bin/../lib/gcc/xtensa-esp-elf/14.2.0/../../../../xtensa-esp-elf/bin/ld: NOTE: This behaviour is deprecated and will be removed in a future version of the linker
Sketch uses 957503 bytes (30%) of program storage space. Maximum is 3145728 bytes.
Global variables use 90156 bytes (27%) of dynamic memory, leaving 237524 bytes for local variables. Maximum is 327680 bytes.
```

other times, the same code failed!

i tried experimenting with different `callconv` expressions, using `addTranslateC` in `build.zig`, but nothing seemed to work consistently.
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

in a sense, i admitted defeat. at the time, i worked through the issue with some LLM assistance: after explaining my predicament, an open weight model was able to isolate the issue in the LLVM source code, identify the problem, and to apply a patch which fixed my compilation issues with little prompting.

this turned out to be a [known issue](https://github.com/llvm/llvm-project/issues/190204) which is [fixed upstream](https://github.com/llvm/llvm-project/pull/200132), but not in the `22.x` release that i was targeting. since i was already compiling my own LLVM, i just included an `llvm.patch` in my repository:

```diff
diff --git a/llvm/lib/Target/Xtensa/MCTargetDesc/XtensaTargetStreamer.cpp b/llvm/lib/Target/Xtensa/MCTargetDesc/XtensaTargetStreamer.cpp
index 0ea70cff4..1dbc36f8a 100644
--- a/llvm/lib/Target/Xtensa/MCTargetDesc/XtensaTargetStreamer.cpp
+++ b/llvm/lib/Target/Xtensa/MCTargetDesc/XtensaTargetStreamer.cpp
@@ -90,6 +90,7 @@ void XtensaTargetELFStreamer::emitLiteral(MCSymbol *LblSym, const MCExpr *Value,

     MCSection *ConstSection = Context.getELFSection(
         SectionName, ELF::SHT_PROGBITS, ELF::SHF_EXECINSTR | ELF::SHF_ALLOC);
+    ConstSection->setAlignment(Align(4));

     OutStreamer.pushSection();
     OutStreamer.switchSection(ConstSection);
@@ -112,6 +113,7 @@ void XtensaTargetELFStreamer::startLiteralSection(MCSection *BaseSection) {
       SectionName, ELF::SHT_PROGBITS, ELF::SHF_EXECINSTR | ELF::SHF_ALLOC);

   ConstSection->setAlignment(Align(4));
+  getStreamer().switchSection(ConstSection);
 }

 MCELFStreamer &XtensaTargetELFStreamer::getStreamer() {
diff --git a/llvm/test/MC/Xtensa/directive-literal.s b/llvm/test/MC/Xtensa/directive-literal.s
index 269cf20ed..1dd9c252d 100644
--- a/llvm/test/MC/Xtensa/directive-literal.s
+++ b/llvm/test/MC/Xtensa/directive-literal.s
@@ -21,6 +21,7 @@ test_literal:

 # CHECK-LITERAL: Section {
 # CHECK-LITERAL:   Name: .literal
+# CHECK-LITERAL:   AddressAlignment: 4
 # CHECK-LITERAL:   SectionData (
 # CHECK-LITERAL:     0000: 78563412 00000000 30750000
 # CHECK-LITERAL:   )
```

recompiling llvm and zig with this patch resolved my fixup woes :)

# exploring the zig object file

i added a small section to `build.zig` so that we could inspect the object file:
```zig
const objectFile = b.step("object-file", "Compile zig object file");
objectFile.dependOn(&b.addInstallFile(
    app.getEmittedBin(),
    "obj/zig_app.o",
).step);
```

this uses the default installation directory, `zig-out`:
```
~/code/zig-inkplate: file zig-out/obj/zig_app.o
zig-out/obj/zig_app.o: ELF 32-bit LSB relocatable, Tensilica Xtensa, version 1 (SYSV), with debug_info, not stripped
```

that looks correct! with `xtensa-esp32-elf-objdump` (which came with my `soldered-inkplate-boards` installation), we can even see the assembly:

```objdump
zig-out/obj/zig_app.o:     file format elf32-xtensa-le


Disassembly of section .text:

    00000000 <zig_app_loop>:
   0:	004136        	entry	a1, 32
   3:	207110        	or	a7, a1, a1
   6:	ffff81        	l32r	a8, 4 <zig_app_loop+0x4> (ff812071 <Target.xtensa.cpu.esp32+0xff811d11>)
			6: R_XTENSA_SLOT0_OP	.literal
   9:	64a0a2        	movi	a10, 100
   c:	00a0b2        	movi	a11, 0
   f:	0008e0        	callx8	a8
  12:	ffffa1        	l32r	a10, 10 <zig_app_loop+0x10> (ffa10008 <Target.xtensa.cpu.esp32+0xffa0fca8>)
			12: R_XTENSA_SLOT0_OP	.literal+0x4
  15:	ffff81        	l32r	a8, 14 <zig_app_loop+0x14> (ffff81ff <Target.xtensa.cpu.esp32+0xffff7e9f>)
			15: R_XTENSA_SLOT0_OP	.literal+0x8
  18:	0008e0        	callx8	a8
  1b:	ffff81        	l32r	a8, 18 <zig_app_loop+0x18> (810008e0 <Target.xtensa.cpu.esp32+0x81000580>)
			1b: R_XTENSA_SLOT0_OP	.literal+0xc
  1e:	0008e0        	callx8	a8
  21:	f01d      	retw.n
	...

00000024 <zig_app_setup>:
  24:	004136        	entry	a1, 32
  27:	207110        	or	a7, a1, a1
  2a:	ffffa1        	l32r	a10, 28 <zig_app_setup+0x4> (ffa12071 <Target.xtensa.cpu.esp32+0xffa11d11>)
			2a: R_XTENSA_SLOT0_OP	.literal+0x10
  2d:	ffff81        	l32r	a8, 2c <zig_app_setup+0x8> (ffff81ff <Target.xtensa.cpu.esp32+0xffff7e9f>)
			2d: R_XTENSA_SLOT0_OP	.literal+0x14
  30:	0008e0        	callx8	a8
  33:	ffff81        	l32r	a8, 30 <zig_app_setup+0xc> (810008e0 <Target.xtensa.cpu.esp32+0x81000580>)
			33: R_XTENSA_SLOT0_OP	.literal+0x18
  36:	0008e0        	callx8	a8
  39:	ffff81        	l32r	a8, 38 <zig_app_setup+0x14> (ffff8100 <Target.xtensa.cpu.esp32+0xffff7da0>)
			39: R_XTENSA_SLOT0_OP	.literal+0x1c
  3c:	0008e0        	callx8	a8
  3f:	f01d      	retw.n

Disassembly of section .literal:

00000000 <.literal>:
	...
			0: R_XTENSA_32	display_set_cursor
			4: R_XTENSA_32	.rodata.str1.1
			8: R_XTENSA_32	display_print
			c: R_XTENSA_32	display_display
			10: R_XTENSA_32	.rodata.str1.1+0xc
			14: R_XTENSA_32	serial_print
			18: R_XTENSA_32	display_begin
			1c: R_XTENSA_32	display_clear 
```

we also see the relocation records that we expect in our `.literal` section, with references to the external functions that we're calling:
```
~/code/zig-inkplate: ~/Library/Arduino15/packages/soldered-inkplate-boards/tools/esp-x32/2511/bin/xtensa-esp32-elf-objdump --reloc -j .literal zig-out/obj/zig_app.o

zig-out/obj/zig_app.o:     file format elf32-xtensa-le

RELOCATION RECORDS FOR [.literal]:
OFFSET   TYPE              VALUE
00000000 R_XTENSA_32       display_set_cursor
00000004 R_XTENSA_32       .rodata.str1.1
00000008 R_XTENSA_32       display_print
0000000c R_XTENSA_32       display_display
00000010 R_XTENSA_32       .rodata.str1.1+0x0000000c
00000014 R_XTENSA_32       serial_print
00000018 R_XTENSA_32       display_begin
0000001c R_XTENSA_32       display_clear
```

this is great!

# running `arduino-cli` from `build.zig`

i'm still using `arduino-cli` to compile and upload my code, but i'll be supplying my zig object file to arduino with the `build.extra_libs` option that my board defines in `boards.txt`:
```
Inkplate4TEMPERA.build.f_cpu=240000000L
Inkplate4TEMPERA.build.flash_size=4MB
Inkplate4TEMPERA.build.flash_freq=80m
Inkplate4TEMPERA.build.flash_mode=qio
Inkplate4TEMPERA.build.boot=dio
Inkplate4TEMPERA.build.partitions=huge_app
Inkplate4TEMPERA.build.custom_partitions=huge_app
Inkplate4TEMPERA.build.defines=-DBOARD_HAS_PSRAM -mfix-esp32-psram-cache-strategy=memw
Inkplate4TEMPERA.build.extra_libs=
Inkplate4TEMPERA.build.memory_type=qio_qspi
```

in `build.zig`, that's a regular function, which takes our `*std.Build.Step.Compile` as input, and specifies the compilation step as a dependency in the build DAG with `dependOn`:
```zig
fn arduinoCompile(
    b: *std.Build,
    app: *std.Build.Step.Compile,
    upload: bool,
    port: ?[]const u8,
) *std.Build.Step {
    const cmd = b.addSystemCommand(&.{
        "arduino-cli",
        "compile",
        "--fqbn",
        "soldered-inkplate-boards:esp32:Inkplate4TEMPERA:UploadSpeed=115200",
        "--output-dir",
        "build",
        "arduino",
    });
    if (upload) {
        cmd.addArgs(&.{"--upload"});
        if (port) |p| cmd.addArgs(&.{ "-p", p });
    }
    cmd.addPrefixedFileArg(
        "--build-property=build.extra_libs=",
        app.getEmittedBin(),
    );
    cmd.step.dependOn(&app.step);
    return &cmd.step;
}
```

this function supports compiling *and* uploading the firmware (which just means adding the `--upload` flag to `arduino-cli` and ensuring that the port is defined).

i'm using `addSystemCommand` to generate a step which executes `arduino-cli`, using the same `std.Build.Step.Compile.getEmittedBin()` function that we called earlier.

with the `--verbose` flag, we can see every command expanded:
```bash
~/code/zig-inkplate: ~/code/zig/build/stage3/bin/zig build --verbose firmware
info(verbose): /Users/jstrickland/code/zig/build/stage3/bin/zig translate-c --cache-dir .zig-cache --global-cache-dir /Users/jstrickland/.cache/zig -target xtensa-freestanding-none arduino/wrapper.h --listen=-
info(verbose): /Users/jstrickland/code/zig/build/stage3/bin/zig build-obj -Odebug -target xtensa-freestanding-none -mcpu esp32 --dep inkplate -Mroot=zig/demo.zig -Odebug -target xtensa-freestanding-none -mcpu esp32 -Minkplate=.zig-cache/o/00e12ac7c5dbf7a53b0c396b9e778995/wrapper.zig --cache-dir .zig-cache --global-cache-dir /Users/jstrickland/.cache/zig --build-root . --name zig_app --zig-lib-dir /Users/jstrickland/code/zig/build/stage3/lib/zig/ --listen=-
info(verbose): install -C .zig-cache/o/ea79841169e767eb0e7e077f49e5da16/zig_app.o zig-out/obj/zig_app.o
info(verbose): arduino-cli compile --fqbn soldered-inkplate-boards:esp32:Inkplate4TEMPERA:UploadSpeed=115200 --output-dir build arduino --build-property=build.extra_libs=./.zig-cache/o/ea79841169e767eb0e7e077f49e5da16/zig_app.o
...
```

- `zig translate-c` to translate `wrapper.h`,
- `zig build-obj` to build the object file,
- `install -C` to copy the object file to `zig-out/obj/zig_app.o`, and
- `arduino-cli compile` with `--build-property=build.extra_libs` set!

# examining the firmware

i'm using the `--output-dir build` argument with `arduino-cli`, so there's a lot of build artifacts leftover for us to explore:

```
~/code/zig-inkplate: file build/arduino.ino.elf
build/arduino.ino.elf: ELF 32-bit LSB executable, Tensilica Xtensa, version 1 (SYSV), statically linked, with debug_info, not stripped
```

instead of a relocatable, it's an executable!

there are a LOT of functions in this binary, but we can locate our zig functions with `xtensa-esp32-elf-objdump -df build/arduino.ino.elf | grep '<zig_app_loop>:' -A 12`:
```objdump
400dcdc0 <zig_app_loop>:
400dcdc0:	004136        	entry	a1, 32
400dcdc3:	207110        	or	a7, a1, a1
400dcdc6:	cede81        	l32r	a8, 400d0940 <_stext+0x920> (400d27d8 <display_set_cursor>)
400dcdc9:	64a0a2        	movi	a10, 100
400dcdcc:	00a0b2        	movi	a11, 0
400dcdcf:	0008e0        	callx8	a8
400dcdd2:	cedca1        	l32r	a10, 400d0944 <_stext+0x924> (3f401bf8 <__anon_1059>)
400dcdd5:	cedc81        	l32r	a8, 400d0948 <_stext+0x928> (400d27e8 <display_print>)
400dcdd8:	0008e0        	callx8	a8
400dcddb:	cedc81        	l32r	a8, 400d094c <_stext+0x92c> (400d27bc <display_display>)
400dcdde:	0008e0        	callx8	a8
400dcde1:	f01d      	retw.n
```

if you look closely, you can see that the previous `l32r` instructions have become concrete!

here's what we had before (`zig_app.o`), with intermediate instructions removed for clarity:
```objdump
   6:	ffff81        	l32r	a8, 4 <zig_app_loop+0x4> (ff812071 <Target.xtensa.cpu.esp32+0xff811d11>)
			6: R_XTENSA_SLOT0_OP	.literal
  12:	ffffa1        	l32r	a10, 10 <zig_app_loop+0x10> (ffa10008 <Target.xtensa.cpu.esp32+0xffa0fca8>)
			12: R_XTENSA_SLOT0_OP	.literal+0x4
  15:	ffff81        	l32r	a8, 14 <zig_app_loop+0x14> (ffff81ff <Target.xtensa.cpu.esp32+0xffff7e9f>)
			15: R_XTENSA_SLOT0_OP	.literal+0x8
  1b:	ffff81        	l32r	a8, 18 <zig_app_loop+0x18> (810008e0 <Target.xtensa.cpu.esp32+0x81000580>)
			1b: R_XTENSA_SLOT0_OP	.literal+0xc
```

(in this case, the `l32r` targets are placeholders, with `R_XTENSA_SLOT0_OP` being their eventual true binding)

and here's our final firmware (`arduino.ino.elf`), when the object file `zig_app.o` has been compiled away:
```objdump
400dcdc6:	cede81        	l32r	a8, 400d0940 <_stext+0x920> (400d27d8 <display_set_cursor>)
400dcdd2:	cedca1        	l32r	a10, 400d0944 <_stext+0x924> (3f401bf8 <__anon_1059>)
400dcdd5:	cedc81        	l32r	a8, 400d0948 <_stext+0x928> (400d27e8 <display_print>)
400dcddb:	cedc81        	l32r	a8, 400d094c <_stext+0x92c> (400d27bc <display_display>)
```

we can see the relocations resolve their references:
- `R_XTENSA_SLOT0_OP .literal+0x0` refers to `<display_set_cursor>`
- `R_XTENSA_SLOT0_OP .literal+0x4` refers to `<__anon_1059>`
- `R_XTENSA_SLOT0_OP .literal+0x8` refers to `<display_print>`
- `R_XTENSA_SLOT0_OP .literal+0xC` refers to `<display_display>`

this matches the offsets in our relocation record from `zig_app.o` exactly:
```
RELOCATION RECORDS FOR [.literal]:
OFFSET   TYPE              VALUE
00000000 R_XTENSA_32       display_set_cursor
00000004 R_XTENSA_32       .rodata.str1.1
00000008 R_XTENSA_32       display_print
0000000c R_XTENSA_32       display_display
00000010 R_XTENSA_32       .rodata.str1.1+0x0000000c
00000014 R_XTENSA_32       serial_print
00000018 R_XTENSA_32       display_begin
0000001c R_XTENSA_32       display_clear
```

# running it on the inkplate

i suppose i haven't shown you the implementation of the C++ wrapper, `wrapper.cpp`! it's very simple:
```c++
#include "wrapper.h"
#include <Arduino.h>
#include <Inkplate.h>

static Inkplate display(INKPLATE_1BIT);

extern "C" {
void display_begin() {
  display.begin();
  display.frontlight.setState(true);
  display.frontlight.setBrightness(5);
  display.setTextSize(2);
  display.setTextColor(BLACK);
}
void display_display() { display.display(); }
void display_clear() { display.clearDisplay(); }
void display_set_cursor(int x, int y) { display.setCursor(x, y); }
void display_print(const char *text) { display.print(text); }
void serial_print(const char *text) { Serial.print(text); }
}
```

our `arduino.ino` is even smaller, and just calls `zig_app_setup` and `zig_app_loop`, which are using the `wrapper.cpp` implementation in our compiled firmware:
```c++
#include <Arduino.h>
#include <Inkplate.h>

extern "C" void zig_app_setup();
extern "C" void zig_app_loop();

void setup() {
  Serial.begin(115200);
  zig_app_setup();
}

void loop() { zig_app_loop(); }
```

i feel comfortable writing this amount of C++.

thank you for reading so far! while it's not the most exciting, here is a picture of "hello world" from zig running on the e-ink hardware:

{{ <figure src="/img/zig_inkplate_hello_world.png" alt="picture of the e-ink screen running our compiled firmware showing 'hello world'"/> }}

the code discussed is available on github:
- https://github.com/djanatyn/zig-inkplate

i haven't ported anything to zig just yet, but knowing that i can fills me with a deep sense of satisfaction, power, and ambition.

i'm only at the start of my journey with zig!
