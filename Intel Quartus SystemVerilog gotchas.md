	% Intel Quartus SystemVerilog gotchas
	% Marek Pikuła
	% 18.09.2019

# Intel Quartus SystemVerilog gotchas

In [official documentation](https://www.intel.com/content/www/us/en/programmable/quartushelp/18.1/index.htm#hdl/vlog/vlog_list_sys_vlog.htm) of Intel Quartus Prime 18.1 it is stated that:

> Intel® Quartus® Prime synthesis supports the following Verilog HDL language standards:
> 
> - SystemVerilog-2005 (IEEE Standard 1800-2005)
> - SystemVerilog-2009 (IEEE Standard 1800-2009)

It is also stated which sections from the specification are supported and which are not.
But as you might expect it is not entirely true.

Of course no one expects a tool to support all language structures and syntax, but amount and severity of non-supported or misinterpreted constructs is staggering.

In this document I would like to express my discontent with Intel Quartus SystemVerilog support and ways to work around the gotchas I stumbled upon while porting [PULPissimo](https://github.com/pulp-platform/pulpissimo/) SoC system to Quartus.

PULPissimo project originally utilizes Xilinx Vivado suite for synthesis and the design has been synthesized with Synopsys toolchain as well.
They don't really have support for any Altera/Intel products whatsoever.

My goal was to port the code, which synthesized beautifully on Xilinx Vivado, to work on Altera Cyclone V on Terasic DE10-Nano board.
To my discontent the toolchain didn't support the code base and it reported errors for various unsupported syntax constructs.

To my greater discontent after successful synthesis the design didn't work, although *the same exact code* worked like a charm with Xilinx Vivado.
This resulted in countless hours spent on trying to get it working.
It resulted in loads of patches, which worked around different incompatibilities.

This document is currently work in progress since the porting isn't finished yet, but the author wanted to make a catalog of all the little things he stumbled upon for future reference, while working on the code.

I hope that provided examples will make lives easier for those of us, who are porting some SystemVerilog code from for example Xilinx Vivado to Intel Quartus or are working on making vendor-independent code.

### Documentation structure

All mentioned projects are submodules of this repo, so that the reader can see the code base and working code.
It is divided into `<name>-base` and `<name>-patched` repos for easy comparison between project trees.

As pointed in Quartus' manual, SystemVerilog specification sections will be referenced according to *IEEE Std 1800-2009 IEEE Standard for System Verilog Unified Hardware Design, Specification, and Verification Language*, which this tool is supposed to support.
All unsupported features, which the documentation is claiming to support are indicated in the first paragraph of an issue section as quoted table excerpt.

Then there is an excerpt from IEEE standard specification and list of all points that are not supported in a way the standard states it.

The final section is a simple example with reference to a file with an unsupported structure.

### License

This work is licensed under the Creative Commons Attribution 4.0 International License.
To view a copy of this license, visit <http://creativecommons.org/licenses/by/4.0/> or send a letter to Creative Commons, PO Box 1866, Mountain View, CA 94042, USA.

Copy of this license is also attached in this repository in file `LICENSE`.

## Synthesis errors

Things that cause synthesizer to show error message and fail synthesis.
If the error message is relevant it is attached in description as well.

### 6.19 Enumerations

#### Quartus documentation

In general enumerations are supported.
There are only some minor quirks in integer value expressions elaborated in _Unsupported features_ section.

> | 6.19 | Enumerations | Supported |
> |-|--|-----|

Quartus might report it as error [10355](https://www.intel.com/content/www/us/en/programmable/quartushelp/18.1/index.htm#msgs/msgs/evrfx_sv_enum_encoded_value_width_mismatch.htm).

#### IEEE standard

> The integer value expressions are evaluated in the context of a cast to the `enum` base type.
> Any enumeration encoding value that is outside the representable range of the `enum` base type shall be an error.

#### Unsupported features

- Integer value expressions are not evaluated in the context of a cast to the enum base type.
  If no width is given it is assumed that given value is a 32-bit integer.
  Solution to this issue is to explicitly state the width of constant to `enum` base type width.

#### Example

From `riscv-dbg/src/dm_pkg.sv`.

Non-compatible code:
```SystemVerilog
typedef enum logic [2:0] {  CmdErrNone, CmdErrBusy, ...,
                            CmdErrorBus, CmdErrorOther = 7
                         } cmderr_e;
```

Compatible code:
```SystemVerilog
typedef enum logic [2:0] {  CmdErrNone, CmdErrBusy, ...,
                            CmdErrorBus, CmdErrorOther = 3'd7
                         } cmderr_e;
```

### 11.4.13 Set membership operator

#### Quartus documentation

To Quartus' credit it is said to be not supported:

> | 11.4.13 | Set membership | Not supported |
> |-|--|-----|

#### IEEE standard

> SystemVerilog supports singular value sets and set membership operators.

#### Unsupported features

There are three basic cases of set membership:

1. Specific values – `if (a inside {b, c})` can be replaced with simple `if ((a == b) || (a == c))`.
2. Ranges – `if (a inside {[b:c]})` can be replaced with simple `if ((a >= b) && (a <= c))`.
3. Variants thereof – combine both.

### 12.5.4 Set membership case statement

#### Quartus documentation

Since [11.4.13](#11413-set-membership-operator) is not supported it can be easily deduced that 12.5.4 won't be supported either.

> | 12.4-12.5 | Selection statement | Supported (unique/priority supported only on case statements) |
> |-|--|-----|

#### IEEE standard

> The keyword `inside` can be used after the parenthesized expression to indicate a set membership (see 11.4.13).

#### Unsupported features in Quartus

- `case inside` construct.

There is no easy way to support all cases.
Here are some possibilities to replace this construct depending on underlying code:

1. No ranges – `inside` keyword is obviously not required and can be removed.
2. Ranges aligned to whole bits – can be replaced with `casez` and don't care values for ranges.
   For example `[3'b000:3'b011]` can be converted to `3'b0??`.
3. Ranges not aligned to whole bits – can be either replaced with `casez` with general wildcard for given range and then inside the case further compared, or it can be translated to `if ... else if ...` construct with ranges, if parallel behaviour is not required (so no `unique` or `parallel` keyword).

### 27.4 Loop generate constructs (×3)

#### Quartus documentation

No reference to section 27.

#### IEEE standard

> A loop generate construct permits a generate block to be instantiated multiple times using syntax that is similar to a for loop statement.
> The loop index variable shall be declared in a `genvar` declaration prior to its use in a loop generate scheme. (…)
>
> Generate blocks in loop generate constructs can be named or unnamed (…)

#### Unsupported features in Quartus

- loop generate itself – must be enclosed in `generate ... endgenerate` block,
- `genvar` inside for loop definition – must be declared outside for loop,
- unnamed loop generate blocks – must be named.

#### Example

From `ibex/ibex_alu.sv`.

Non-compatible code:
```SystemVerilog
for (genvar k = 0; k < 32; k++) begin
  assign operand_a_rev[k] = operand_a_i[31-k];
end
```

Compatible code:
```SystemVerilog
generate
genvar k;
for (k = 0; k < 32; k++) begin : gen_rev_operand_a
  assign operand_a_rev[k] = operand_a_i[31-k];
end
endgenerate
```

### 27.5 Conditional generate constructs

#### Quartus documentation

No reference to section 27.

Quartus might report it as error [10170](https://www.intel.com/content/www/us/en/programmable/quartushelp/18.1/index.htm#msgs/msgs/evrfx_veri_syntax_error.htm).

#### IEEE standard

> The conditional generate constructs, if-generate and case-generate, select at most one generate block from a set of alternative generate blocks based on constant expressions evaluated during elaboration.
> The selected generate block, if any, is instantiated into the model.

There is no mention of mandatory `generate ... endgenerate`, although in some examples it is nowhere to be found.

#### Unsupported features in Quartus

- conditional generate without `generate ... endgenerate` block – must be enclosed in `generate ... endgenerate`.

#### Example

From `ibex/ibex_core.sv`.

Non-compatible code:
```SystemVerilog
if (PMPEnable) begin : g_pmp
  ...
end else begin : g_no_pmp
  ...
end
```

Compatible code:
```SystemVerilog
generate
if (PMPEnable) begin : g_pmp
  ...
end else begin : g_no_pmp
  ...
end
endgenerate
```

### Double semicolon

It's not particularly bad thing of Quartus to point out.
It doesn't like double semicolons at the end of line.

Quartus might report it as error [10170](https://www.intel.com/content/www/us/en/programmable/quartushelp/18.1/index.htm#msgs/msgs/evrfx_veri_syntax_error.htm).

## Synthesis gotchas

Things that don't cause synthesizer to show error message, but they synthesize in unexpected way.

### 12.5.2 Constant expression in case statement

#### Quartus documentation

> | 12.4-12.5 | Selection statement | Supported (unique/priority supported only on case statements) |
> |-|--|-----|

#### IEEE Standard

> A constant expression can be used for the *case_expression*.
> The value of the constant expression shall be compared against the *case_item_expressions*.

#### Unsupported features in Quartus

The simple `case (1'b1)` for priority flag assertion or similar application usually don't work.
Sometimes they do, but it's quiet unpredictable.
Replace it with `if ... else if ... else`.

#### Example

From `ibex/ibex_cs_registers.sv`.

Non-compatible code:
```SystemVerilog
unique case (1'b1)
  csr_save_if_i: begin
    exception_pc = pc_if_i;
  end
  csr_save_id_i: begin
    exception_pc = pc_id_i;
  end
  default:;
endcase
```

Compatible code:
```SystemVerilog
if (csr_save_if_i) begin
  exception_pc = pc_if_i;
end if (csr_save_id_i) begin
  exception_pc = pc_id_i;
end
```

### `case` defaults sometimes not working

## Ways of verifying which parts of code don't synthesize

When starting to port code after successfully fixing all errors as described in [Synthesis errors](#synthesis-errors) section you might have some specific block not working properly.
To find out what's wrong you might want to review RTL if it looks OK and, to Quartus' credit, its RTL Viewer is really functional (although the author finds Vivado's RTL viewer more informative for code verification).

In this section you can find a few things to be alert about when doing RTL review for unwanted synthesis optimizations.

### Unused pins

First thing to do when there is a problem with given instance is to look at its input and output pins.
If they are used in code and you can see that in Vivado they are used in RTL (so they are not optimized out in first stage) they should be used in Quartus as well.
It's very simple step, but can save a lot of time while identifying the most obvious problems.

The way to do it is find a problematic instance in either *Netlist Navigator* or with search functionality and poke at all input pins to see if they drive any logic and if they drive subjectively enough logic in comparison for Vivado RTL.

### Filter node sources

If given output node should be driven by some other nodes depending on case, it should be visible in RTL.
The hard way is to trace all the wires in full view.
The easy way is to right click on the problematic node and select *Filter→Sources* (or *Shift+S*).

This is particularly useful for verifying `case` statements.
