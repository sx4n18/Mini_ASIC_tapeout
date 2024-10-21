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


## 29 Aug 2024

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



#### High-Speed Serial Transceiver

There is a map that shows the count number of the high-speed transceiver count numbers by different products.

![Total transceiver count and the transmission speed](./img/7-series-FPGA-high-speed-serial-transceiver-count.png)

Attached are some user guide to use [GTP transceiver](https://docs.amd.com/v/u/en-US/ug482_7Series_GTP_Transceivers) and [GTX/GTH transceiver](https://docs.amd.com/v/u/en-US/ug476_7Series_Transceivers)

Here is another map of aggregate bandwidth:

![Here lies the transceiver aggregate bandwidth, not sure about the unit](./img/transceiver-aggregate-bandwidth-not-sure-the-unit.png)


And finally, there is a figure that shows the total I/O count and total bandwidth in Gb/s 

![I/O count and total bandwidth calculated using LVDS performance](./img/Total_IO_count_and_bandwidth_7_series_FPGA.png)


In the Artix-7 product brief, it was mentioned that the double differential I/O standards with speed up to **1.25 Gb/s** LVDS performance

After checking these details, I think I know how these numbers are calculated.

For a GTP transceiver with max speed of 6.6 Gb/s,

Transceiver aggregate bandwidth = 2 X number of transceiver X 6.6

Therefore, XC7A12T, which has 2 GTP transceiver, has 2 X 2 X 6.6 = 26.4 Gb/s.

It has to multiply by 2 because we are counting the duplex working bandwidth.

Total bandwidth simply just used 150 X 1.25 = 187.5 Gb/s


#### AC701 Evaluation board kit of Artix-7

The board I have been checking is [AC701 evaluation board](https://docs.amd.com/v/u/en-US/ug952-ac701-a7-eval-bd), which ships with the XC7A200T


The AC701 board provides access to eight GTP transceivers:

+ Four of the GTP transceivers are wired to the PCI Express x4 endpoint edge connector (P1) fingers
+ Two of the GTP transceivers are wired to the FMC HPC connector (J30)
+ One GTP transceiver is wired to SMA connectors (RX: J46, J47 TX: J44, J45)
+ One GTP transceiver is wired to the SFP/SFP+ Module connector (P3)


## 30 Aug 2024

Now I am restarting the implementation in the virtuoso environment innovus.

Where I invoke innovus in the directory where OA libraries are accessible.

This is a better way to implement the design for later virtuoso import.

Somehow the tool reports that there is no physical data for certain cells.


I chose TECH_XH018_HD as the technology library instead of TECH_XH018.

I am wondering if this could be the problem. I will now switch to TECH_XH018 instead.


**PROBLEM SOLVED!**


Technology library should be **TECH_XH018** instead of **TECH_XH018_HD**.

This made me wonder if I have made a mistake by using HD tech lef file instead of the normal lef file.

But this should not impact the synthesis, because we did not do physical synthesis iSpatial flow.

Now it is reporting something that feels like unresolvable, where it complains the special net VDD/VSS distance is too close.


![A DRC error that complains about distance required between VDD and VSS needs to be at least 0.6](./img/DRC_error_that_feels_unresolvable_on_IP.png)

But this cannot be fixed if we have this distance set up by the RAM IP block.


I am thinking if it is because I chose 1143_HD when generating the RAM IP block.


Also I just realised that the supporting power rail only has to be thicker than half of the Wxfmc.


And also considering the limited ports, I shall regenerate another FIFO DPRAM with width of 8 and restart the implementation.

And I also realised that the IP has a technology code as in 1133??

Which is not what we are using if my memory serves.

Our design should be 

**MET4+METMID+METTHK**

not 

**MET3+METMID+METTHK**


Now I am regenerating a new FIFO RAM!!! It should have the process code of 1143 and MET4+METMID+METTHK.

Also it will only have 128 entries and 8 bit wide.

I will also generate another 256 entries and 8 bit wide.

I also left the load capacitance alone this time with default 0.2pF.

In terms of variation, I am creating the one that's closest to a square.


### Start over again with 128X8 FIFO

I have just drafted the verilog module again for this FIFO, simply change some parameters and that's it.

This will be extended to next week to complete.



## 02 Sep 2024


Since I have already drafted the verilog for 128X8 FIFO, now I just have to do another round of simulation to check out the behaviour.

As expected, the behaviour is correct.

Now I will head to the Synthesis again with 125 degrees.

Synthesis finished without any error.

Now doing the conformal check...

conformal check passed...

Now heading to place and route...

draft up the MMMC file and constraint files...

OK this MMMC file finally passed...

Moving the work to the virtuoso environment for PnR...

Dont know why I am having this error again about the physcial cells not found??

reload innovus again and this is fixed?

Now I have just did a special routing, and this is the least DRC I have ever seen.

![Least DRC erros I have ever seen after SRoute of a FIFO design](./img/least_DRC_I_have_seen_after_sRoute_FIFO_128_8.png)

Now I guess I need to fix these errors manually



## 03 Sep 2024

### Fixing DRC erros brought by via generation

There are loads of errors brought by the via generation they are either too close to each other.

Mainly at the via between metal 4 and metal 5, the tool used a fixed via cell to place on top of each other, this will inevitably make the vias overlap or leave no space between vias.

I tried to create my own customised via which seems to be ok to address this issue.

![Newly generated via between metal 4 and 5](./img/customised_via_between_metal5_metal_4_to_remove_DRC.png)

I drew a horizontal metal 4 that is equally thick and manually added 5 vias (4 on the left and 1 on the right) and made sure they satisfy the DRC rule.

And the DRC error typically looks like this:

![Typical Via 4 error in this design](./img/typical_via_4_DRC_error_to_be_addressed.png)

After I fixed it manually, the layout looks like this:

![The 3D view after I manually fixed the error](./img/typical_via_4_DRC_error_after_manual_fix.png)

I have just fixed all the DRC errors in my design.

Now I am doing cell placement.

The tool used Metal 6 over the RAM block, but this has not been finalised and there are routes overlapped.

and apparenlty, the RAM module has an area that is not allowed for routing namingly OBS.

![congestion map after early global route](./img/congestion_map_after_early_global_route_mainly_vertical.png)

After checking the details on virtuoso, it looks like the routing blockage for metal 6 is tiny, and the routing we have in innovus should actually be ok.

![routing blockages found for metal 6 in virtuoso is a tinay purple area](./img/only_a_tiny_area_that_blocks_METAL_6_so_it_should_be_ok.png)


Finished clock tree synthesis, now 2 buffers added for the clock tree distribution.

we still have hold time violation.

Now we are optimising the route for hold violation.

And then the violation is fixed, it looks the tool inserted some buffers.


## 04 Sep 2024


I will continue the place and route from yesterday following the clock tree synthesis and hold timing violation fix.

Suprisingly, routing finished pretty quickly with only minor errors.

And there is no routing above the block using other than metal 6.

There is no DRC errors but 9 connectivity errors mainly in 2 catelogues:

+ Antenna side area ratio 
+ dangling wires of the power stripes.

After trimming the unnecessary power stripes, the dangling wires violations are all fixed except the antenna side area ratio violation.

I will have a look online and see if there is any tutorial.

So this [article](https://chipedge.com/what-is-the-antenna-effect-in-vlsi/#:~:text=Antenna%20Violation%20occurs%20when%20the,the%20conductor%20(gate%20area).) talks about what the antenna ratio is and possible solutions.

There is another [article](https://www.edn.com/antenna-violations-resolved-using-new-method/) talking more comprehensively about the solution and causes.

So basically, antenna ratio is the ratio of gate area and gate oxide area.

and the solutions it proposed includes:

+ Metal jumper
+ Diode insertion

Metal jumper will break the signal line and use jumpers to route them on a top metal.

The second solution will attach the diode to the wire and establish a discharge path


### What is Antenna effect

It is also known as plasma-induced gate-oxide damage or plasma-induced damage.

It occurs when unwanted charges accumulate on exposed conductors during fabrication.


I will proceed from now, cus there is no DRC error and connectivity error.

The design will be saved 


### Fixing antenna violation

I therefore followed metal hopping method to fix the antenna hopping.

Before the metal hopping, the violation exists for signal CLK.

![CLK signal antenna error before the fix](./img/Antenna_error_needs_to_be_fixed_for_CLK_signal.png)


After I broke the signal CLK net and did a little wire editing, the antenna violation is fixed.

![Antenna hopping is fixed after metal hopping](./img/Fix_antenna_violation_by_adding_metal_hopping_CLK.png)


## 05 Sep 2024


I shall give the implemented design a post-layout simulation and conformal verification today.

Ok, it looks like the design passed the behaviour check...

![Post layout gate level simulation presented](./img/looks_like_the_post_layout_simulation_is_working_for_FIFO128.png)

It is worth noting that because the highest bit of register write_ptr is at the end of the scan chain, it was therefore replaced by scan_out during optimisation.

Surpsingly it works.... and the simulation also works even if I did not type in the Dual port ram verilog file???

I am still scratching my head about this...


Now let me do the conformal check for the design...

It it reporting 20 nonequivalent DFFs...

![The tool is reporting 20 nonequivalent DFFs but I do not see the difference between these 20 nonequivalence](./img/conformal_check_results_of_syn_and_PnR_showing_slight_difference_but_I_do_not_see_non_equivalence.png)


I should now probably check how conformal works.


## 09 Sep 2024

I guess I can probably do another FIFO implementation with 256 entries of DPRAM

Behaioural simulation passed !

post-synthesis simulation passed !

LEC conformal check passed, except the DB input not mapped. This is beacuase in the original code, they are not connected to anything. During synthesis, it has been all connected to ground.

In conclusion, LEC check passed !

PnR now... this shall take some time.


## 10 Sep 2024

I have managed to push the design forward to finish the placement.

This also includes clock tree synthesis, but it seems the placement preference has been down at the bottom of the RAM.

Basically all the cells are placed under the RAM.

![FIFO implementation 256 ](./img/FIFO_256_Cell_placements_after_clock_tree_synthesis.png)


And this got me thinking I should probably start over with a different floorplan, I mean the side areas are basically not used.


## 11 Sep 2024

I have finished the design for FIFO 256, will start testing tomorrow.

The final layout looks like this:

![Final layout of the finished 256 FIFO](./img/Finished_256_FIFO_design_layout.png)

But I am having a lot of concerns for this design.

for example, I did not reorder the scan chain to make the placement more efficient.


## 12 Sep 2024

Just checked the 128 FIFO design and realised there are still some connectivity errors need fixing.

Will come back to this tomorrow, and if necessary, I will start over with the place and route.



## 13 Sep 2024

I am now coming back to the reimplementation of the 128 FIFO after I realised how messed up the last implementation was.

And I am also getting the scan chain reordered to have a better placement.

And I am also connecting the DB of the RAM to the ground.

Just redid the implementation of the FIFO 128 after checking and fixed everything.

this has now been saved in Sync_FIFO_XDPRAM_128X8 as FIFO_128_redo.

Will get the functionality verified next


## 14 Sep 2024

Since I have got the implementation finished, I will now get the functionality verified. 

Oh and also get this conformal checked.

I just learnt that while writing out the SDF files, you can put the options of PVT informations like follow:

And according to innovus, it is recommended to turn on recompute_delaycal for functionality simulation.

It is normally turned off for faster computation.

```Tcl
 write_sdf -process best:typical:worst -voltage 1.98:1.80:1.62 -temperature -40:25:125 -recompute_delaycal FIFO_128_8_XDPRAM/PnR/outputs/FIFO_128_redo_final.sdf  
```

And the functionality has been verified to be ok.

![functionality verified FIFO 128 redo ](./img/FIFO_128_redo_functionality_verified_to_be_working.png)


Also I noticed that the tool complained about the timing declaration missing from key verilog files, which has never happened before.


I will head on to the conformal check.

This has reported the similar situmation as before, so I would assume it is the behaviour that might exist commonly with innovus.

Even though I have not verified the scan chain from the design.

Continuing with the 256 implementation, I see that I should add tie high and tie low cells before placing the cell.

But there are 2 types of tie hi/lo cells, with with low voltage optimisation, one without.

I will stick to the normal one again.

And also this time, I set a placment blockgage so that the cells will not be place under the power stripes.

While adding fillers, I noticed the error messages saying the site has been defined as "HD":

```text
**ERROR: (IMPSP-365):	Design has inst(s) with SITE 'core_hd_dh', but the floorplan has no rows defined for this site. Any locations found for such insts will be illegal; create rows for this site to avoid this.
```

I think it is complaining about the double height fillers because of the field defined and therefore, there are no double height filler cells inserted.

of course there are antenna violations...

And due to the right side of the RAM block not aligned with the edge, there are fillers cells placed, and this has caused basically every row's power rail to be open.... shiiiiiit.

Let me start over again....

But because reallocate the placement for the RAM is going to bring extra more work, I will just add a placement halo to the right.

Now I have finished FIFO 256 reimplementation too, I will probably head on for further up integration.


## 17 Sep 2024

Today I can get this physcial implementation verified probably using simulation and/or conformal.

The behaviour looks correct except the signal almost_full is a little glitchy but it should not be a problem.

![The FIFO 256 simulation looks correct from the waveform](./img/behavioural_simulation_at_the_post_layout_gate_level_simulation_of_FIFO_256.png)

and I would assume the confomal check still cannot pass because of the slightly rearranged DFFs.

Of course, as expected, there are non-equivalent DFFs 

But I am going to go ahead and ignore this for now...

There are couple points I may want to revisit again in the future.

It seems that I am wastig an extra cycle here at reading out process according to the simulation.

![The data has already been ready one clock cycle ahead](./img/double_counted_output_at_QB_and_it_flips_along_with_the_clock.png)

I also need to consider carefully about the chip level integration now.

Like, how this data loop should look like and how many pins I need.


## 23 Sep 2024

Think I have a plan for the whole chip design. 

Because what we have in mind is very simple, just a simple data loop.

Serial/parallel in --> parallisation --> FIFO --> serialisation --> serial out

And this time, I would like to use 512 rows of DPRAM.

About the wasted extra cycle, it is clearly illustrated from the behavioural simulation that there is no extra cycle wasted.

And the behaviour I drafted will make sure the correct data can be exported.


Steve's email makes me realise my concern is valid, like this IP purchase may not be economical, which might only be used once.

Because we might need to switch technology later on.

But the RAM should be able to fit in.


a 256 words 8 bit DPRAM is 337.11μm x 210.61μm

And 512 words 8 bit DPRAM is 339.79μm x 297.97μm

The die size for this miniasic should be 1520μm x 1520μm


256X8

337.11 X 210.61

512X8

339.79 X 297.97

FIFO imp:

422.49 X 284.87


I aslo finished the RTL for input handler, which should be hooked to the FIFO's input.

Will implement the output handler tomorrow, which should take the FIFO's output and then serialise it to SPI protocol.


## 24 Sep 2024

Just sent the email confirming the block we are using this time is 512X8.

Since I have finished the input handler, now I will proceed to the output handler to serialise the output.


I have found an SPI implementation repo, and now I am reading all of it to understand the implementation better.


## 25 Sep 2024

After asking chapGPT, I think my understanding towards the SPI interface is clearer now.

I understood the mechanism of the clock generation and realised I might need a state machine for my need.

Since I have a process of dealing with the FIFO, I need to let it request data from FIFO.

I have drafted the output handler and simulated the behaivour, it looks alirght from my simulation.

![output handler simulation looks alrigh](./img/output_handler_looks_alright_from_the_simulation_with_proper_SPI_behaviour.png)


## 26 Sep 2024

Just learnt that if you wish to insert breakpoint, the option -linedebug is needed when invoking xcelium simulator.

My plan next may be to find an FPGA and my Arduino board to see if the SPI module works.

Coming up next should be hooking up the FIFO and output handler together to do a global simulation.

I need to probably hook up the input handler first before the output handler.

And never use ```$urandom()```, the simulator is not happy with it.

I have just finished the simulation for the combination of input handler and FIFO module, it shows the behaviour has been correct.

![Expected behaviour for the combination of input handler and FIFO](./img/expected_behaviour_of_input_handler_and_FIFO_showing_correct_writing_and_reading.png)

OK, after observing the waveform, I thinkg I need some control over the output logic, because the statemachine has been stuck in the position where it is waiting for the read data from the FIFO after asserting the RD_n.

![modification needed for output handler module](./img/some_control_signal_is_needed_to_control_the_reading_from_FIFO.png)

I will fix this tomorrow.


## 27 Sep 2024

I was thinking towards the issue I had, the possible solution I can think of is:

+ make the output handler not output anything if empty/almost_empty flagged
+ add an extra signal for output control

Think it is easier to make the statemachine check the flag status of empty/almost_empty

![Added empty flag signal helpped the output handler](./img/added_empty_flag_worked_with_simulation_for_output_handler.png)

I will now add this port in the full data loop and make the connection.

Think the behaviour is now fixed.

![The simulation looks ok after the solution implemented](./img/full_data_loop_behaivour_fixed_in_the_simulation_this_is_looking_well.png)

Ok, this is easier than I thought...

But I am still not sure if the SPI is behaving correctly.

Will have to get this verified with an SPI IP in vivado.


## 30 Sep 2024

I think I could probably use vivado IP to verify the implementation of my SPI module.

After checking other people's design, I think my SPI should work just fine (probably needs further verification).


I have just finished the synthesis of the design, now I need to do the conformal check.

Even though reported some non-equivalence, I think they should be equivalent, this is because the syntehsis has optimised my redundant logic with Wr_en and data_received.

I will now do the post-synthesis simulation to verify at least the behaviour is ok.

And it looks that the post-synthesis simulation presented correct behaviours.

![post-synthesis simulation with actual gate presented correct behaivour with the full data loop](./img/post_syn_gate_level_simulation_proves_to_be_working.png)


## 01 Oct 2024

After the meeting with Piotr, I was told I have another job to do.

Try my best to make the design work at 200MHz...

I will first have to double the clock rate apparently, and now I will have to make a multi-mode chip.

Think I have a rough plan:

1. I will need to make sure the whole design works at 100MHz
2. Depending on the functional mode, the output handler will be operating either at 100MHz or 200 MHz


## 02 Oct 2024

Despite what has been brought to me, I will make an implementation to make sure we have something to deliver first.

It would be very irrational to change everything now.

Start drafting MMMC file now.

OK, I did not add reset synchronisation, I will need to add it now after the compilation for the MMMC file has passed.

Mannually added reset synchronisation looks like this:

```verilog

wire rst_n_sync_1, rst_n_sync_2;

// reset synchronisation
//  (C, D, Q, RN);
DFRRQHDX1 reset_sync_1 ( 
	.C(CLK), 
	.D(1'b1), 
	.Q(rst_n_sync_1), 
	.RN(rst_n));

DFRRQHDX1 reset_sync_2 ( 
	.C(CLK), 
	.D(rst_n_sync_1), 
	.Q(rst_n_sync_2), 
	.RN(rst_n));
```

Now I need to make changes at the sdc files, which has been made.


Apparenlty, there is a thing called routing rule for clock, and generally people would use double width double spacing rule for clock :

First, you need to create a non-default routing rule (NDR): 

```tcl 
create_route_rule -name CTS_2W2S -spacing_multiplier {MET3:METTPL 2} -width_multiplier {MET3:METTPL 2}
```

Then create a routing type for CTS 

```tcl 
create_route_type -name cts_trunk -route_rule CTS_2W2S \
			-top_preferred_layer MET4   -bottom_preferred_layer MET3 \
			-shield_net VSS   -bottom_shield_layer MET3
```

Finally apply property to trunk type clock nets 

```tcl 
set_db cts_route_type_trunk cts_trunk 
```

I will come back for the DRC and antenna fix tomorrow



## 03 Oct 2024

I will finish the design today.

First I checked the conectivity violations, which says VDD is open.

After checking, it appears to be the issue from filler cells near the corner of the RAM cell where I was not expecting cells.

So their power rail was missing and I then simply deleted the cells.

Then I did some metal hopping to fix antenna violations.

Now the design is properly finished.

But then I found that post-route hold violation was not fixed....

it is now very difficult to fix with everything filled....

So I restarted from post_cts_fixed where filler cells have not been inserted and this time the tool magiacally inserted diodes to fix antenna violations.

And it is smart enough to do metal hopping to fix antenna, amazing!!!

![Finally inserted a diode to fix antenna violation!](./img/Finally_successfully_inserted_a_diode_to_fix_antenna_violations.png)

The only antenna violation found in this case is an easy fix.

Now I will check the timing.

Of course there are hold violations and easily fixable with buffer cells.

![Easilfy fixable hold timing violation when filler cells are not placed.](./img/Easily_fixable_hold_timing_violation_post_route.png)

Now I can add filler cells.

```tcl 
add_fillers -base_cells FEED25HD FEED15HD FEED10HD FEED7HD FEED5HD FEED3HD FEED2HD FEED1HD FCNE32HD FCNE31HD FCNE30HD FCNE29HD FCNE28HD FCNE27HD FCNE26HD FCNE25HD FCNE24HD FCNE23HD FCNE22HD FCNE21HD FCNE20HD FCNE19HD FCNE18HD FCNE17HD FCNE16HD FCNE15HD FCNE14HD FCNE13HD FCNE12HD FCNE11HD FCNE10HD FCNE9HD FCNE8HD FCNE7HD FCNE6HD FCNE5HD FCNE4HD FCNE3HD FCNE2HD DECAP25HD DECAP15HD DECAP10HD DECAP7HD DECAP5HD DECAP3HD -prefix FILLER_
```
Think I need to write a proper flow guide for this technology so that I can always follow properly.


Now I need to verify the post-layout gate level behaviour.

OK, I think the functionality has been verified from the simulation.

![functionality has been verified through the post-layout simulation](./img/Final_gate_level_simulation_results_post_layout.png)

And I should also verify the scan chain probably.

![The simulation test for scan chain, it looks like it works](./img/the_scan_chain_looks_functional_in_this_simulation.png)

And from the simulation, it looks like there are 53 registers in total.

It would be best if I can keep the whole design in digital, otherwise I need another set of VDD and GND.

We do not have that many pins.

I will come back tomorrow for the final layout.


## 04 Oct 2024

Ok, let's finish this layout.

I have finished the layout and did a PVS DRC check and it has passed except an antenna violation.

## 07 Oct 2024

Still the power net has not been connected, I will try to connect it mannually.

I still do not feel confident to plot out the power and ground connection.

I should probably email Steve and ask for a favour.

I cannot fix the design on virtuoso, I am so not familiar with it...



## 14 Oct 2024

I will reimplement the design today start from floorplan and rename the power ground and probably add the clock tree with shield net.

Shouyu section V2.0 starting now.

This is the following useful script to set up the spacing and non default rules.-help

```tcl 
add_ndr -name default_2x_space -spacing {metal1 0.38 metal2:metal5 0.42 metal6 0.84}
create_route_type -name leaf_rule  -non_default_rule default_2x_space -top_preferred_layer metal4 -bottom_preferred_layer metal2
create_route_type -name trunk_rule -non_default_rule default_2x_space -top_preferred_layer metal4 -bottom_preferred_layer metal2 -shield_net VSS -shield_side both_side
create_route_type -name top_rule   -non_default_rule default_2x_space -top_preferred_layer metal4 -bottom_preferred_layer metal2 -shield_net VSS -shield_side both_side
set_ccopt_property route_type -net_type leaf  leaf_rule
set_ccopt_property route_type -net_type trunk trunk_rule
set_ccopt_property route_type -net_type top   top_rule
```


and because the clock is gonna use metal 3 - metal 5, here I am gonna use some important information:

```lef
LAYER MET3
    TYPE ROUTING ;
    WIDTH 0.28 ;
    MAXWIDTH 35 ;
    SPACING 0.28 ;
    SPACING 0.6 RANGE 10.001 100 ;
    SPACING 0.6 RANGE 10.001 100 INFLUENCE 1.6 ;
    PITCH 0.610 ;
    OFFSET 0.000 ;
    AREA 0.202 ;
    WIREEXTENSION 0.19 ;
    DIRECTION HORIZONTAL ;
    CAPACITANCE CPERSQDIST 4.15e-05 ;
    EDGECAPACITANCE 8.3e-06 ;
    RESISTANCE RPERSQ 0.074 ;
    THICKNESS 0.565 ;
    ANTENNASIDEAREARATIO 380 ;
    ANTENNADIFFAREARATIO 9999999 ;
END MET3

LAYER MET4
    TYPE ROUTING ;
    WIDTH 0.28 ;
    MAXWIDTH 35 ;
    SPACING 0.28 ;
    SPACING 0.6 RANGE 10.001 100 ;
    SPACING 0.6 RANGE 10.001 100 INFLUENCE 1.6 ;
    PITCH 0.630 ;
    OFFSET 0.315 ;
    AREA 0.202 ;
    WIREEXTENSION 0.19 ;
    DIRECTION VERTICAL ;
    CAPACITANCE CPERSQDIST 4.15e-05 ;
    EDGECAPACITANCE 8.3e-06 ;
    RESISTANCE RPERSQ 0.074 ;
    THICKNESS 0.565 ;
    ANTENNASIDEAREARATIO 380 ;
    ANTENNADIFFAREARATIO 9999999 ;
END MET4

LAYER METTP
    TYPE ROUTING ;
    WIDTH 0.44 ;
    MAXWIDTH 35 ;
    SPACING 0.46 ;
    SPACING 0.6 RANGE 10.001 100 ;
    SPACING 0.6 RANGE 10.001 100 INFLUENCE 1.6 ;
    PITCH 1.220 ;
    OFFSET 0.610 ;
    AREA 0.562 ;
    DIRECTION HORIZONTAL ;
    CAPACITANCE CPERSQDIST 3.53e-05 ;
    EDGECAPACITANCE 9.5e-06 ;
    RESISTANCE RPERSQ 0.031 ;
    THICKNESS 0.985 ;
    ANTENNASIDEAREARATIO 380 ;
    ANTENNADIFFAREARATIO 9999999 ;
END METTP
```

Based on this I wrote the following:

```tcl 
## create non default routing rule 

create_route_rule -name double_spacing_of_default_3_5 -spacing {MET3 0.56 MET4 0.56 METTP 0.92}

## create_routing type 
create_route_type -name leaf_rule -route_rule double_spacing_of_default_3_5 -top_preferred_layer METTP -bottom_preferred_layer MET3

create_route_type -name trunk_rule -route_rule double_spacing_of_default_3_5 -top_preferred_layer METTP -bottom_preferred_layer MET3 -shield_net gnd! -shield_side both

create_route_type -name top_rule -route_rule double_spacing_of_default_3_5 -top_preferred_layer METTP -bottom_preferred_layer MET3 -shield_net gnd! -shield_side both


## connect the routing type to the cts routing property

set_db cts_route_type_leaf leaf_rule

set_db cts_route_type_trunk trunk_rule

set_db cts_route_type_top top_rule
``` 

![CTS double spacing and shielding worked](./img/Messy_but_the_cts_double_spacing_rule_worked.png)

But there would be a lot of problems for DRC afterwards, I think it is because the space was not enough


## 15 Oct 2024

Today I will be trying to set up the whole flow in innovus.



## 17 Oct 2024

This time I found the options to insert the map file during GDS export and they can also be done using scripts:

```tcl 
write_stream outputs/GDS/Double_encoder -map_file /eda/design_kits/xkit_root/xh018/cadence/v9_0/PDK/IC61/v9_0_4/TECH_XH018_HD_1131/pnr_streamout.map -lib_name DesignLib -unit 1000 -mode all
```

And now I shall try to import this GDS back into virtuoso and see if it will be successful.

So far it seems that the configurations have been very helpful and all the layers are shown in virtuoso.

But in the meantime, it is also creating a lot of via cells.


I think it is because the tool created a new via structure to make sure it can be displayed in virtuoso.

For example:

encoder_VIA0 is a cell with 2 vias on top of each other.

![encoder_VIA0 a new cell created with 2 vias on top of each other](./img/GDSII_imported_with_extra_vias_new_cell.png)

and it seems that TECH_1131 AND TECH_1131_HD does not have very big differences.

