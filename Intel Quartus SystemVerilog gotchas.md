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

As pointed in Quartus' manual, SystemVerilog specification sections will be referenced according to *IEEE Std 1800-2009 IEEE Standard for System Verilog Unified Hardware Design, Specification, and Verification Language*, which this tool is supposed to support.