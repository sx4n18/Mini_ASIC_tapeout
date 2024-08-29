# Mini ASIC chip tape out aka project LANA

This is the log for mini ASIC tape out project to implement the FIFO with embedded RAM IP.

This project is named project _**LANA**_

## 23 Aug 2024

I have reviewed some papers about LVDS and high-speed interface and feel that this might be a tricky topic to address at the moment. This is because it involves both digital and analogue design technique to achieve and probably level shifting.


So I wanted to drop this topic just now and proceed with the IP implementation flow for now with the newly generated 6 metal RAM block.

I will start again with synchronous FIFO implementation.

And I want to establish an automatic flow for synthesis and place and route along this way so that I can just rerun when I need to.


First I will have to organise the Verilog implementation again and set this up functionally.

Since I have drafted the behavioural model for sync FIFO, I will just use the version 1.0 and make small tweaks on it.

Apart from the old header, we will use an additional ready signal to indicate the read data is available. This is a new feature I added on asynchronous FIFO.

This signal will be asserted when data is ready until the new ready signal is asserted.

Also the behaviour for flagging almost full and almost empty has been updated with a more comprehensive logic where

```Verilog
assign almost_full = (!empty) && ((read_ptr + FIFO_size - write_ptr) % FIFO_size <= threshold);
assign almost_empty = (write_ptr + FIFO_size - read_ptr) % FIFO_size <= threshold;
```

almost full may be incorrectly triggered when read_ptr == write_ptr and therefore should be added with one extra condition of not empty.

Now I shall start the synthesis stage.


### Synthesis

And it seems that so long as you load the liberty file and LEF file into the tool, you do not need the Verilog file of the IP to work.

And I am setting the clock period at 50MHz for synthesis using constraint file, now the sdc file is a minimal file.

This one will have a scan path through the design.

Synthesis finished, it was fairly quick.

Now I can probably run a post-synthesis simulation?

Just ran a post-synthesis simulation, and it appears that the writing process has not been impacted, but the reading process is a little problematic.

somehow, the internal read_delay was not updating itself.

Think there might have been a timing issue that I have missed.

Will check the log now.

![read delay signal is not counting](./img/internal_register_not_updating_leading_to_failing_read_out.png)


I will now rerun the synthesis and specifically check out what read_delay has been connected to.

And also somehow the synthesis decided the most significant bit of write_ptr is not connected and left at high-impedance.

So far the behaviour looks very odd, and I will come back to this next week.


## 27 Aug 2024

I shall go back to the synthesis and see if the problem is reproducible and probably have a good look at the schematic, or send the design through conformal check.


I will send the design through conformal check first.

### Conformal LEC check

The LEC conformal check report back that two designs pre/post synthesis are equivalent except the black box.

This has got me wonder what has happened during synthesis.

It probably means that the logic has been redundant and pruned.

![The running result from LEC conformal check](./img/conformal_check_of_the_FIFO_design_showing_strong_equivalence.png)

### Redo Synthesis

now the synthesis redo has been finished, I do not spot any apparent errors or warnings that worth noting.

Will have to rerun simulation again.

And I am using the general digital library Verilog file to do the simulation.


Ok, there are warnings talking about the timing annotation failed on the non-existent paths, which means that the digital library Verilog is not the right one for simulation.

But this simulation shows correct behaviour.

Will now do the simulation again with the library with timing information.

The problem was reproduced, not sure what the problem is, I will try to extend the read_enable timing.

I extended the period of the read enable from 20 to 40 and then the simulation worked.

![The setup timing issue was solved by asserting the read_enable for 40 ns](./img/read_enable_assereted_4ns_prior_2_posedge_clk.png)

I think it is because of the setup timing violation encountered during simulation, this is because the read_enable has only been asserted low 4 ns prior to the positive edge of the clock.

Now I will try to assert the read_enable near the negative edge of the clock and then re run the simulation.

And now it works miraculously well.

![The reading now works well with the timing in place to leave enough time to setup](./img/Simulation_works_well_when_read_enable_is_asserted_further_away_from_possedge.png)

After couple testing, it could be concluded that the assertion should be at least 7 ns ahead of the posedge of the clock.

Since I have checked the behaviour of the design and it is correct, I will now move on and start the PnR implementation.


### Place and Route

I will now need to write up the MMMC file 😭.



## 28 Aug 2024

### Place and Route (continued)

I drafted the MMMC file with 3 different analysis view where they are described below:

From the document, the power supply for the IP should be 20 um wide at least.

So I have tried to power the design with 20 um power stripes, in the meantime, I will also add a power ring around the RAM block.


## 28 Aug 2024

### Place and Route (continued-continued)

To avoid dangling wires along each rows, I extended the margin left at the left and right side.

This has allow me to leave two big power rails at the margin area so that the metal 1 can be connected.

Also I can add the ring first and then add the stripe to allow it to be merged with the block ring wherever possible.

I placed the cells and there is only very very little areas used for actual cell placement.

![The area actually needed for cell placement is insane](./img/very_little_cells_in_a_vast_big_ass_floorplan.png)


There are couple of problems that need to be addressed.

+ There are dangling wires of the cell power rails, these are mainly because there are no via connections made on the block ring.
+ There are feedthroughs on the RAM block. (Potentially, this is not a problem)
+ The tie high and tie low signals on the RAM blocks are not connected.
+ The utilisation rate in this floor plan is disastrous. 


I will need to address these in my next version of application.


In the meantime, I need to search for the FPGA LVDS availability and how fast they can go.


### FPGA LVDS support search

I will mainly check Xilinx board 7 series at the moment for the I/O support.

According to the product selection guide listed [here](https://docs.amd.com/v/u/en-US/7-series-product-selection-guide), the I/O resources listed on an FPGA are listed below in the table.

But it is worth noting that not **ALL** ports can be fully used.


#### Artix-7

There are different FPGA parts with the I/O resources listed in the table below:

| Part number   | XC7A12T | XC7A15T | XC7A25T  | XC7A35T | XC7A50T | XC7A75T | XC7A100T | XC7A200T |
| ------------- |---------| --------| ---------| --------|---------|---------|----------|----------|
|Max single-ended I/O|150 | 250 | 150 | 250 | 250 | 300 | 300 | 500|
|Max differential pair| 72| 120 | 72  | 120 | 120 | 144 | 144 | 240|



#### Kintex-7

| Part number   | XC7K70T | XC7K160T | XC7K325T  | XC7K355T | XC7K410T | XC7K420T | XC7K480T | 
| ------------- |---------| --------| ---------| --------|---------|---------|----------|
|Max single-ended I/O|300 | 400 | 500 | 300 | 500 | 400 | 400 |
|Max differential pair| 144| 192 | 240  | 144 | 240 | 192 | 192 |



#### High-Speed Serial Transceiver Count

There is a map that shows the count number of the high-speed transceiver count numbers by different products.

![Total transceiver count and the transmission speed](./img/7-series-FPGA-high-speed-serial-transceiver-count.png)

Attached are some user guide to use [GTP transceiver](https://docs.amd.com/v/u/en-US/ug482_7Series_GTP_Transceivers) and [GTX/GTH transceiver](https://docs.amd.com/v/u/en-US/ug482_7Series_GTP_Transceivers)


