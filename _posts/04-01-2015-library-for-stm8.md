---
layout: default
title: FLOSS C library for STM8 microcontroller 
---
Today I create initial code (some headers) for [libstm8](https://github.com/mnd/libstm8/)
library for STM8 microcontrollers.

For now this headers contains only STM8L registers defines and defines for
interrupt numbers. This library would be tested with 
[STM8L-Discovery](http://www.st.com/stm8l-discovery) debug board,
FLOSS [SDCC](http://sdcc.sourceforge.net/) compiler and 
[stm8flash](https://github.com/vdudouyt/stm8flash) utility to load compiled 
code to STM8L-Discovery boart through st-link interface. 

In next week I will continue work with describing stm8l registers and write
some examples.
