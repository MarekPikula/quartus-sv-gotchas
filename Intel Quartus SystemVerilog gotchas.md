	% Intel Quartus SystemVerilog gotchas
	% Marek Pikuła
	% 18.09.2019

# Intel Quartus SystemVerilog gotchas

In [official documentation](https://www.intel.com/content/www/us/en/programmable/quartushelp/18.1/index.htm#hdl/vlog/vlog_list_sys_vlog.htm) it is stated that:

> Intel® Quartus® Prime synthesis supports the following Verilog HDL language standards:
> 
> - SystemVerilog-2005 (IEEE Standard 1800-2005)
> - SystemVerilog-2009 (IEEE Standard 1800-2009)

Of course no one expects a tool to support all language structures and syntax. It is also stated which sections from the specification are supported and which are not. BUT it is not entirely true as you will see in next sections.

In this document I would like to express my discontent with Intel Quartus SystemVerilog support and ways to work around the gotchas I stumbled upon, while porting [PULPissimo](https://github.com/pulp-platform/pulpissimo/) SoC system to Quartus.

PULPissimo project originally uses Xilinx Vivado suite for synthesis and the design has been synthesized with Synopsys toolchain as well. They don't really have support for any Altera/Intel products whatsoever.

My goal was to port the code, which synthesized beautifully on Xilinx Vivado, to work on Altera's Cyclone V on Terasic's DE10-Nano board. To my discontent the toolchain didn't support entire codebase and reported errors for various unsupported syntax constructs. To my greater discontent after successful synthesis the design didn't work, although *the same exact code* worked like a charm with Xilinx Vivado. This resulted in countless hours trying to get it working. It resulted in loads of patches, which worked around different incompatibilities and somewhat working project.

All mentioned projects are submodules of this repo, so that the reader can see the code base and working code. It is divided into `<name>-base` and `<name>-patched` repos for easy comparison between project trees.

This document is currently work in progress since the porting isn't finished yet, but the author wanted to make a catalog of all the little things he stumbled upon for future reference, while working on the code.

I hope that provided examples will make lives easier for those of us, who are porting some SystemVerilog code from for example Xilinx Vivado to Intel Quartus.

As pointed in Quartus' manual, SystemVerilog specification sections will be referenced according to *IEEE Std 1800-2009 IEEE Standard for System Verilog Unified Hardware Design, Specification, and Verification Language*, which this tool is supposed to support. All unsupported features, which the documentation is claiming to support are indicated in the first paragraph of an issue section as quoted table excerpt.

This work is licensed under the Creative Commons Attribution 4.0 International License. To view a copy of this license, visit <http://creativecommons.org/licenses/by/4.0/> or send a letter to Creative Commons, PO Box 1866, Mountain View, CA 94042, USA.

## Synthesis errors

Things that cause synthesizer to show error message.

### 27.4 Loop generate constructs (×3)

**Quartus documentation:**

No reference to section 27.

**IEEE standard:**

> A loop generate construct permits a generate block to be instantiated multiple times using syntax that is similar to a for loop statement. The loop index variable shall be declared in a `genvar` declaration prior to its use in a loop generate scheme. (…)
>
> Generate blocks in loop generate constructs can be named or unnamed (…)

**Unsupported features in Quartus:**

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

### Other `generate` blocks

### `unique case inside` not supported

## Synthesis gotchas

Things that don't cause synthesizer to show error message, but they synthesize in unexpected way.

### 12.5.2 Constant expression in case statement

**Quartus documentation:**

> | 12.4-12.5 | Selection statement | Supported (unique/priority supported only on case statements)
> |-|--|-----|

**IEEE Standard:**

> A constant expression can be used for the *case_expression*. The value of the constant expression shall be compared against the *case_item_expressions*.

**Unsupported features in Quartus:**

The simple `case (1'b1)` for priority flag assertion or similar application usually don't work. Sometimes they do, but it's quiet unpredictable. Replace it with `if ... else if ... else`.

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

When starting to port code after successfully fixing all errors as described in [Synthesis errors](#synthesis-errors) section you might have some specific block not working properly. To find out what's wrong you might want to review RTL if it looks OK and, to Quartus' credit, its RTL Viewer is really functional (although the author finds Vivado's RTL viewer more informative for code verification).

In this section you can find a few things to be alert about when doing RTL review for unwanted synthesis optimizations.

### Unused pins

First thing to do when there is a problem with given instance is to look at its input and output pins. If they are used in code and you can see that in Vivado they are used in RTL (so they are not optimized out in first stage) they should be used in Quartus as well. It's very simple step, but can save a lot of time while identifying the most obvious problems.

The way to do it is find a problematic instance in either *Netlist Navigator* or with search functionality and poke at all input pins to see if they drive any logic and if they drive subjectively enough logic in comparison for Vivado RTL.

### Filter node sources

If given output node should be driven by some other nodes depending on case, it should be visible in RTL. The hard way is to trace all the wires in full view. The easy way is to right click on the problematic node and select *Filter→Sources* (or *Shift+S*).

This is particularly useful for verifying `case` statements.
