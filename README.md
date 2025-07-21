# TCL_Workshop
TCL (Tool Command Language) is a versatile scripting language commonly used to automate processes in VLSI design. From design setup to verification, TCL helps streamline workflows, making it a key part of modern EDA tools and custom automation.

## Day 1: Introduction to TCL and VDSYNTH Toolbox Usage
In this workshop, we are building a TCL script (referred to as a TCL box), which takes a CSV file as input and generates timing results as output. 

<img width="1424" height="727" alt="image" src="https://github.com/user-attachments/assets/030d4998-ed86-4216-82b0-13ad32359ac0" />

We are given several tasks to understand how TCL is used for automation - 
1. Create a custom command (ex. vsdsynth) to pass .csv files from the UNIX shell to the TCL script
2. Convert .csv input to format [1] (as required by the TCL script) and SDC (Synopsys Design Constraints) format
3. The converted inputs are passed to Yosys synthesis tool (conversion to format [1] is done as Yosys does not support .csv files directly)
4. Convert the outputs (format [1] + SDC) to format [2] for use with the OpenTimer tool for Static Timing Analysis (STA).
5. Generate a final report showing timing results.

### Task - 1: Create a custom command (ex. vsdsynth) to pass .csv files from the UNIX shell to the TCL script
We need to create a UNIX script (custom command), "vsdsynth". 

1. Informing the system that it's a UNIX script
   ```
   #!/bin/tcsh -f
   ```

2. Creating the logo
   <img width="935" height="693" alt="image" src="https://github.com/user-attachments/assets/4e432aa9-21f7-458a-b683-30c89831b278" />

3. Making the script executable
   ```
   chmod +x vsdsynth
   ```

4. Create a dummy vsdsynth.tcl file
   This is to test if everything connects properly
   ```
   echo 'puts "TCL script executed with file: $argv"' > vsdsynth.tcl
   ```

5. Test the script
   Case 1: No arguments
   ```
   ./vsdsynth
   ```
   <img width="578" height="271" alt="image" src="https://github.com/user-attachments/assets/843efe3c-caee-4934-944b-ef7236886599" />

   Case 2: Non-existent file
   ```
   ./vsdsynth dummy.csv
   ```
   <img width="689" height="270" alt="image" src="https://github.com/user-attachments/assets/3c411c02-b5e1-4e7a-9053-9a40c6c2b63d" />

   Case 3: Help
   ```
   ./vsdsynth -help
   ```
   <img width="599" height="398" alt="image" src="https://github.com/user-attachments/assets/6f0dc211-238a-4831-9b50-ecd6364f33c9" />

   Case 4: Valid CSV file
   Create a dummy CSV:
   ```
   touch test.csv
   ```
   Then run:
   ```
   ./vsdsynth test.csv
   ```
   <img width="657" height="272" alt="image" src="https://github.com/user-attachments/assets/3b3f0fe3-a01d-4352-8bbd-7007aa91d861" />

If you're using bash by default, but want to use tcsh, install it with:
```
sudo apt-get install tcsh  # For Ubuntu/Debian
sudo yum install tcsh      # For Oracle Linux/CentOS
```
You must have tclsh installed. If not:
```
sudo apt-get install tcl
```
   
## Day 2: Variable Creation and Processing Constraints from CSV
### Task - 2: Convert .csv input to format [1] (as required by the TCL script) and SDC (Synopsys Design Constraints) format

1. Setup & Install Required TCL Packages
   ```
   sudo apt update
   sudo apt install tcllib
   ```
   <img width="1638" height="903" alt="image" src="https://github.com/user-attachments/assets/c7eb72cc-9909-43f2-b698-a10e0ed08b62" />

2. Read openMSP430_design_details.csv and Create Variables
   ```
   set filename [lindex $argv 0]
   package require csv
   package require struct::matrix
   struct::matrix m

   set f [open $filename]
   csv::read2matrix $f m , auto
   close $f

   set columns [m columns]
   m link my_arr
   set rows [m rows]
   ```

3. Convert CSV to Variables (Remove Spaces and Normalize Paths)
   ```
   set i 0
   while {$i < $rows} {
      puts "\nInfo: Setting $my_arr(0,$i) as '$my_arr(1,$i)'"
      if {$i == 0} {
        set [string map {" " ""} $my_arr(0,$i)] $my_arr(1,$i)
      } else {
        set [string map {" " ""} $my_arr(0,$i)] [file normalize $my_arr(1,$i)]
      }
      set i [expr {$i+1}]
   }
   ```
   <img width="1212" height="627" alt="image" src="https://github.com/user-attachments/assets/3eba9f7c-a05b-4f98-a02d-4ac9def81872" />
   
4. Print All Initialized Variables
   ```
   puts "\nInfo: Below are the list of the initial variables and their values:"
   puts "DesignName = $DesignName"
   puts "OutputDirectory = $OutputDirectory"
   puts "NetlistDirectory = $NetlistDirectory"
   puts "EarlyLibraryPath = $EarlyLibraryPath"
   puts "LateLibraryPath = $LateLibraryPath"
   puts "ConstraintsFile = $ConstraintsFile"
   ```
   <img width="1163" height="631" alt="image" src="https://github.com/user-attachments/assets/8677581b-2aa5-409e-9259-c85b4c31fe8f" />

5. Check If Paths Exist, Create or Exit Accordingly
   ```
   if { [file isdirectory $OutputDirectory]} {
    puts "\nInfo : Output directory exists and found in path $OutputDirectory "
   } else {
    puts "\nInfo : Cannot find output directory. Creating $OutputDirectory"
    file mkdir $OutputDirectory
   }

   if { [file isdirectory $NetlistDirectory]} {
    puts "\nInfo : Netlist directory exists at $NetlistDirectory "
   } else {
    puts "\nInfo : Cannot find Netlist directory. Creating $NetlistDirectory"
    file mkdir $NetlistDirectory
   } 

   if { ![file exists $EarlyLibraryPath]} {
    puts "\nERROR: Cannot find early library: $EarlyLibraryPath. Exiting..."
    exit
   }

   if { ![file exists $LateLibraryPath]} {
    puts "\nERROR: Cannot find late library: $LateLibraryPath. Exiting..."
    exit
   }

   if { ![file exists $ConstraintsFile]} {
    puts "\nERROR: Cannot find constraints file: $ConstraintsFile. Exiting..."
    exit
   }
   ```
   As shown in the figure, if all the files and directories mentioned exist, then we can go ahead with the next step.
   <img width="1218" height="818" alt="image" src="https://github.com/user-attachments/assets/abe48a09-3788-44a2-b9d8-154a3c2b3925" />

   
6. Convert Constraints CSV into Matrix
   ```
   puts "\nInfo: Dumping SDC constraints for $DesignName"

   ::struct::matrix constraints
   set chan [open $ConstraintsFile]
   csv::read2matrix $chan constraints , auto
   close $chan

   set constr_rows [constraints rows]
   set constr_columns [constraints columns]

   puts "Number of rows in constraints file = $constr_rows"
   puts "Number of columns in constraints file = $constr_columns"
   ```
   <img width="396" height="40" alt="image" src="https://github.com/user-attachments/assets/0e3f319c-607a-4cff-972a-a6401ee16c59" />

   <img width="1449" height="879" alt="image" src="https://github.com/user-attachments/assets/7040eefa-7747-4c54-af6d-e2d1e07f55e8" />


7. Convert to SDC and Find Start Indexes of Clock/Input/Output Constraints
   ```
   set clock_start [lindex [lindex [constraints search all CLOCKS] 0] 1]
   set clock_start_column [lindex [lindex [constraints search all CLOCKS] 0] 0]
   puts "clock_start =  $clock_start"
   puts "clock_start_column = $clock_start_column"

   set input_ports_start [lindex [lindex [ constraints search all INPUTS] 0] 1]
   puts "input_ports_start = $input_ports_start"

   set output_ports_start [lindex [lindex [ constraints search all OUTPUTS] 0] 1]
   puts "output_ports_start = $output_ports_start"
   ```
   <img width="1152" height="858" alt="image" src="https://github.com/user-attachments/assets/ee3fe774-84cc-4df3-9adf-b1a9c238ce76" />

    
## Day 3: Processing clock and input constraints
1. Get clock constraint column indices for delay and slew
```
set clock_early_rise_delay_start  [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$constr_columns -1}] [expr {$input_ports_start -1}] early_rise_delay] 0 ] 0]
set clock_early_fall_delay_start  [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$constr_columns -1}] [expr {$input_ports_start -1}] early_fall_delay] 0 ] 0]
set clock_late_rise_delay_start   [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$constr_columns -1}] [expr {$input_ports_start -1}] late_rise_delay] 0 ] 0 ]
set clock_late_fall_delay_start   [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$constr_columns -1}] [expr {$input_ports_start -1}] late_fall_delay] 0 ] 0 ]
set clock_early_rise_slew_start   [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$constr_columns -1}] [expr {$input_ports_start -1}] early_rise_slew] 0 ] 0 ]
set clock_early_fall_slew_start   [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$constr_columns -1}] [expr {$input_ports_start -1}] early_fall_slew] 0 ] 0 ]
set clock_late_rise_slew_start    [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$constr_columns -1}] [expr {$input_ports_start -1}] late_rise_slew] 0 ] 0 ]
set clock_late_fall_slew_start    [lindex [lindex [constraints search rect $clock_start_column $clock_start [expr {$constr_columns -1}] [expr {$input_ports_start -1}] late_fall_slew] 0 ] 0 ]
```
2. Print the clock column indices
```
puts "clock_early_rise_delay_start = $clock_early_rise_delay_start"
puts "clock_early_fall_delay_start = $clock_early_fall_delay_start"
puts "clock_late_rise_delay_start = $clock_late_rise_delay_start"
puts "clock_late_fall_delay_start = $clock_late_fall_delay_start"
puts "clock_early_rise_slew_start = $clock_early_rise_slew_start"
puts "clock_early_fall_slew_start = $clock_early_fall_slew_start"
puts "clock_late_rise_slew_start = $clock_late_rise_slew_start"
puts "clock_late_fall_slew_start = $clock_late_fall_slew_start"
```
<img width="811" height="832" alt="image" src="https://github.com/user-attachments/assets/f97612a7-0a06-4640-8ea7-9d404759787c" />

3. Open the SDC file for writing the clock constraints
```
set sdc_file [open $OutputDirectory/$DesignName.sdc "w"]
set i [expr {$clock_start+1}]
set end_of_ports [expr {$input_ports_start -1}]
puts "\n Info-SDC: Working on clock constraints..."
```
4. Write clock constraints (create_clock, latency, transition)
```
while {$i < $end_of_ports} {
    puts "Working on clock [constraints get cell 0 $i ]"
    puts -nonewline $sdc_file "\ncreate_clock -name [constraints get cell 0 $i ] -period [constraints get cell 1 $i ] -waveform \{0 [expr { [constraints get cell 1 $i ]*[constraints get cell 2 $i ]/100 }]\} \[get_ports [constraints get cell 0 $i ]\]"
    puts -nonewline $sdc_file "\nset_clock_latency -source -early -rise [constraints get cell $clock_early_rise_delay_start $i ] \[get_clocks [constraints get cell 0  $i]\]"
    puts -nonewline $sdc_file "\nset_clock_latency -source -early -fail [constraints get cell $clock_early_fall_delay_start $i ] \[get_clocks [constraints get cell 0  $i]\]"
    puts -nonewline $sdc_file "\nset_clock_latency -source -late -rise [constraints get cell $clock_late_rise_delay_start $i ] \[get_clocks [constraints get cell 0  $i]\]"
    puts -nonewline $sdc_file "\nset_clock_latency -source -late -fail [constraints get cell $clock_late_fall_delay_start $i ] \[get_clocks [constraints get cell 0  $i]\]"
    puts -nonewline $sdc_file "\nset_clock_transition -rise -min [constraints get cell $clock_early_rise_slew_start $i ] \[get_clocks [constraints get cell 0  $i]\]"
    puts -nonewline $sdc_file "\nset_clock_transition -fall -min [constraints get cell $clock_early_fall_slew_start $i ] \[get_clocks [constraints get cell 0  $i]\]"
    puts -nonewline $sdc_file "\nset_clock_transition -rise -max [constraints get cell $clock_late_rise_slew_start $i ] \[get_clocks [constraints get cell 0  $i]\]"
    puts -nonewline $sdc_file "\nset_clock_transition -fall -max [constraints get cell $clock_late_fall_slew_start $i ] \[get_clocks [constraints get cell 0  $i]\]"
    set i [expr {$i+1}]
}
```
<img width="1219" height="630" alt="image" src="https://github.com/user-attachments/assets/609f0287-85eb-487d-81e7-0dbf4393e002" />

In the figure above, we can see that the clock parameters are successfully written into the .sdc file. We also verified that the values match exactly with those specified in the constraints.csv file.

5. Get input constraint column indices for delay and slew
```
set input_early_rise_delay_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$constr_columns-1}] [expr {$output_ports_start-1}] early_rise_delay] 0] 0]
set input_early_fall_delay_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$constr_columns-1}] [expr {$output_ports_start-1}] early_fall_delay] 0] 0]
set input_late_rise_delay_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$constr_columns-1}] [expr {$output_ports_start-1}] late_rise_delay] 0] 0]
set input_late_fall_delay_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$constr_columns-1}] [expr {$output_ports_start-1}] late_fall_delay] 0] 0]
set input_early_rise_slew_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$constr_columns -1}] [expr {$output_ports_start -1}] early_rise_slew] 0 ] 0 ]
set input_early_fall_slew_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$constr_columns -1}] [expr {$output_ports_start -1}] early_fall_slew] 0 ] 0 ]
set input_late_rise_slew_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$constr_columns -1}] [expr {$output_ports_start -1}] late_rise_slew] 0 ] 0 ]
set input_late_fall_slew_start [lindex [lindex [constraints search rect $clock_start_column $input_ports_start [expr {$constr_columns -1}] [expr {$output_ports_start -1}] late_fall_slew] 0 ] 0 ]
```
6. Print input column indices
```
puts "input_early_rise_delay = $input_early_rise_delay_start"
puts "input_early_fall_delay = $input_early_fall_delay_start"
puts "input_late_rise_delay = $input_late_rise_delay_start"
puts "input_late_fall_delay = $input_late_fall_delay_start"
puts "input_early_rise_slew_start = $input_early_rise_slew_start"
puts "input_early_fall_slew_start = $input_early_fall_slew_start"
puts "input_late_rise_slew_start = $input_late_rise_slew_start"
puts "input_late_fall_slew_start = $input_late_fall_slew_start"
```
<img width="530" height="520" alt="image" src="https://github.com/user-attachments/assets/b71c5e62-9220-4390-a04e-7e2e1259da28" />

7. Check verilog declarations and categorize bussed vs single-port

<img width="1294" height="726" alt="image" src="https://github.com/user-attachments/assets/ecf8b000-688c-441f-a9b3-d4ed376aee66" />

In the figure above, we observe that some input ports are bussed while others are not. However, the constraints.csv file does not indicate which ports are part of a bus and which are individual signals. For better clarity and compatibility with the OpenTimer tool, we need to expand bussed ports into their individual bits. To do this, we must identify whether each input port is a single-bit port or a bus. If it is a bus, we are going to append a '*' to the port name in the .sdc file.

Some of the input ports have multiple spaces between them. The below script removes the spaces between them and writes them in a temporary file. The script reads one of input ports as "input cpu_en" and later it is written in /tmp/1 file as "input cpu_en"
```
set i [expr {$input_ports_start+1}]
set end_of_ports [expr {$output_ports_start-1}]
puts "\nInfo-SDC: Working on IO constraints....."
puts "\nInfo-SDC: Categorizing input ports as bits and bussed"

while {$i < $end_of_ports} {
    set netlist [glob -dir $NetlistDirectory *.v]
    set tmp_file [open /tmp/1 w]
    foreach f $netlist {
        set fd [open $f]
        while {[gets $fd line] != -1} {
            set pattern1 " [constraints get cell 0 $i];"
            if {[regexp -all -- $pattern1 $line]} {
                set pattern2 [lindex [split $line ";"] 0]
                if {[regexp -all {input} [lindex [split $pattern2 "\S+"] 0]]} {
                    set s1 "[lindex [split $pattern2 "\S+"] 0] [lindex [split $pattern2 "\S+"] 1] [lindex [split $pattern2 "\S+"] 2]"
                    puts -nonewline $tmp_file "\n[regsub -all {\s+} $s1 " "]"
                }
            }
        }
        close $fd
    }
    close $tmp_file
```
<img width="1848" height="304" alt="image" src="https://github.com/user-attachments/assets/fc5ee0e9-d6a7-42c3-9b51-41be038972f3" />

8. Check if the port is bussed (based on the number of tokens)
    ```
    set tmp_file [open /tmp/1 r]
    set tmp2_file [open /tmp/2 w]
    puts -nonewline $tmp2_file "[join [lsort -unique [split [read $tmp_file] \n]] \n]"
    close $tmp_file
    close $tmp2_file

    set tmp2_file [open /tmp/2 r]
    set count [llength [read $tmp2_file]]
    if {$count > 2} {
        set inp_ports [concat [constraints get cell 0 $i]*]
    } else {
        set inp_ports [constraints get cell 0 $i]
    }
    ```
    <img width="800" height="977" alt="image" src="https://github.com/user-attachments/assets/513322c2-bc84-4a02-a59d-bbe8e25ae619" />

9. Write input delay and slew values into SDC file
    ```
    puts -nonewline $sdc_file "\nset_input_delay -clock \[get_clocks [constraints get cell $related_clock $i]\] -min -rise -source_latency_included [constraints get cell $input_early_rise_delay_start $i] \[get_ports $inp_ports\]"
    puts -nonewline $sdc_file "\nset_input_delay -clock \[get_clocks [constraints get cell $related_clock $i]\] -min -fall -source_latency_included [constraints get cell $input_early_fall_delay_start $i] \[get_ports $inp_ports\]"
    puts -nonewline $sdc_file "\nset_input_delay -clock \[get_clocks [constraints get cell $related_clock $i]\] -max -rise -source_latency_included [constraints get cell $input_late_rise_delay_start $i] \[get_ports $inp_ports\]"
    puts -nonewline $sdc_file "\nset_input_delay -clock \[get_clocks [constraints get cell $related_clock $i]\] -max -fall -source_latency_included [constraints get cell $input_late_fall_delay_start $i] \[get_ports $inp_ports\]"

    puts -nonewline $sdc_file "\nset_input_transition -clock \[get_clocks [constraints get cell $related_clock $i]\] -min -rise -source_latency_included [constraints get cell $input_early_rise_slew_start $i] \[get_ports $inp_ports\]"
    puts -nonewline $sdc_file "\nset_input_transition -clock \[get_clocks [constraints get cell $related_clock $i]\] -min -fall -source_latency_included [constraints get cell $input_early_fall_slew_start $i] \[get_ports $inp_ports\]"
    puts -nonewline $sdc_file "\nset_input_transition -clock \[get_clocks [constraints get cell $related_clock $i]\] -max -rise -source_latency_included [constraints get cell $input_late_rise_slew_start $i] \[get_ports $inp_ports\]"
    puts -nonewline $sdc_file "\nset_input_transition -clock \[get_clocks [constraints get cell $related_clock $i]\] -max -fall -source_latency_included [constraints get cell $input_late_fall_slew_start $i] \[get_ports $inp_ports\]"

    set i [expr {$i+1}]

   ```
<img width="1243" height="716" alt="image" src="https://github.com/user-attachments/assets/c114ac02-dc44-41f3-9229-fa13a3038d5f" />

# Day 4: Complete Scripting and Yosys Synthesis Introduction
In this module, we will complete the sub-task 2 by converting output constraints into sdc format. We will also do the synthesis by using Yosys tool.

In the previous module we have completed converting input constraints into sdc format. Similarly, do the same process for output parameters. We have to change only some parameters such as setting **i**_as **$output_ports_start** etc..

The script mentioned below finds the column number of the output parameters.
```
set output_early_rise_delay_start [lindex [lindex [constraints search rect $clock_start_column $output_ports_start [expr {$constr_columns-1}] [expr {$constr_rows-1}]  early_rise_delay] 0 ] 0]
set output_early_fall_delay_start [lindex [lindex [constraints search rect $clock_start_column $output_ports_start [expr {$constr_columns-1}] [expr {$constr_rows-1}]  early_fall_delay] 0 ] 0]
set output_late_rise_delay_start [lindex [lindex [constraints search rect $clock_start_column $output_ports_start [expr {$constr_columns-1}] [expr {$constr_rows-1}]  late_rise_delay] 0 ] 0]
set output_late_fall_delay_start [lindex [lindex [constraints search rect $clock_start_column $output_ports_start [expr {$constr_columns-1}] [expr {$constr_rows-1}]  late_fall_delay] 0 ] 0]
set output_load_start [lindex [lindex [constraints search rect $clock_start_column $output_ports_start [expr {$constr_columns-1}] [expr {$constr_rows-1}]  load] 0 ] 0]
set related_clock [lindex [lindex [constraints search rect $clock_start_column $output_ports_start [expr {$constr_columns-1}] [expr {$constr_rows-1}]  clocks] 0 ] 0]
puts "output_early_rise_delay_start =$output_early_rise_delay_start"
puts "output_early_fall_delay_start =$output_early_fall_delay_start"
puts "output_late_rise_delay_start =$output_late_rise_delay_start"
puts "output_late_fall_delay_start =$output_late_fall_delay_start"
puts "output_load_start =$output_load_start"
```
We can find the output parameters column numbers in the below figure.
<img width="852" height="633" alt="image" src="https://github.com/user-attachments/assets/21c38406-39ac-4921-86e6-e12e9dc27840" />


We have to develop an alogrithm to check if the output ports are bussed or not. If bussd, concat "*" at the end of port name. Then write the output parameters into .sdc format in a particular sdc foramt. The script mentioned below does the above purpose.

```
set i [expr {$output_ports_start+1}]
set end_of_ports [expr {$constr_rows-1}]
puts "\nInfo-SDC: Working on Output constraints....."
puts "\nInfo-SDC: Categorizing output ports as bits and bussed"

while { $i < $end_of_ports } {

set netlist [glob -dir $NetlistDirectory *.v]
set tmp_file [open /tmp/1 w]

foreach f $netlist {
        set fd [open $f]
		#puts "reading file $f"
        while {[gets $fd line] != -1} {
			set pattern1 " [constraints get cell 0 $i];"
            if {[regexp -all -- $pattern1 $line]} {
			#puts "\npattern1 \"$pattern1\" found and matching line in verilog file \"$f\" is \"$line\""
				set pattern2 [lindex [split $line ";"] 0]
			#puts "\ncreating pattern2 by splitting pattern1 using semi-colon as delimiter => \"$pattern2\""
				if {[regexp -all {outptu} [lindex [split $pattern2 "\S+"] 0]]} {	
			#puts "\nout of all patterns, \"$pattern2\" has matching string \"input\". So preserving this line and ignoring others"
				set s1 "[lindex [split $pattern2 "\S+"] 0] [lindex [split $pattern2 "\S+"] 1] [lindex [split $pattern2 "\S+"] 2]"
				#puts "\nprinting first 3 elements of pattern as \"$s1\" using space as delimiter"
				puts -nonewline $tmp_file "\n[regsub -all {\s+} $s1 " "]"
				#puts "\nreplace multiple spaces in s1 by space and reformat as \"[regsub -all {\s+} $s1 " "]\""
				}
				#else { " \"$pattern2\" didnt have first term as 'output'"}
        	}
        }
close $fd
}
close $tmp_file

set tmp_file [open /tmp/1 r]
set tmp2_file [open /tmp/2 w]
#puts "reading [read $tmp_file]"
#puts "reading /tmp/1 file as  [split [read $tmp_file] \n]] ]"
#puts "sorting /tmp/1 file as [lsort -unique [split [read $tmp_file] \n]]"
#puts "joining /tmp/1 file as [join [lsort -unique [split [read $tmp_file] \n]] \n]"
puts -nonewline $tmp2_file "[join [lsort -unique [split [read $tmp_file] \n]] \n]"
close $tmp_file
close $tmp2_file
set tmp2_file [open /tmp/2 r]

set count [llength [read $tmp2_file]] 
#puts "Count is $count"
if {$count > 2} { 
    set op_ports [concat [constraints get cell 0 $i]*]
	#puts "\n Bussed"
} else {

    set op_ports [constraints get cell 0 $i]
	#puts "\n Not Bussed"
}
	#puts "output port name is $inp_ports and count is $count"

	puts -nonewline $sdc_file "\nset_output_delay -clock  [constraints get cell $related_clock $i] -min -rise  [constraints get cell $output_early_rise_delay_start $i] \[get_ports $op_ports\]"
        puts -nonewline $sdc_file "\nset_output_delay -clock  [constraints get cell $related_clock $i] -min -fall  [constraints get cell $output_early_fall_delay_start $i] \[get_ports $op_ports\]"
        puts -nonewline $sdc_file "\nset_output_delay -clock  [constraints get cell $related_clock $i] -max -rise  [constraints get cell $output_late_rise_delay_start $i] \[get_ports $op_ports\]"
        puts -nonewline $sdc_file "\nset_output_delay -clock  [constraints get cell $related_clock $i] -max -fall  [constraints get cell $output_late_fall_delay_start $i] \[get_ports $op_ports\]"
		puts -nonewline $sdc_file "\nset_load [constraints get cell $output_load_start $i] \[get_ports $op_ports\]"



	set i [expr {$i+1}]

}
puts "\n Info:SDC Created .Please use the constraints path $OutputDirectory/$DesignName.sdc"
close $tmp2_file

close $sdc_file
```

We can see the output parameters are written in sdc format in .sdc file 

After the successful completion of writng all constraints parameters into .sdc file, we get a message saying "SDC created.Please use the constraints path /home/vsduser/SynthTime/outdir_openMSP430/$openMSP430.sdc" 

#### Creating Scripts for Hierarchy check


```

puts "\n Info: Creating hierarchy check script to be used by Yosys"
set data "read_liberty -lib -ignore_miss_dir -setattr blackbox ${LateLibraryPath}"
#puts "data is \"$data\""
set filename "$DesignName.hier.ys"
#puts "\nfilename is \"$filename\""
set fileId [open $OutputDirectory/$filename "w"]
#puts "open \"$OutputDirectory/$filename"\ in write mode"
puts -nonewline $fileId $data
#puts "netlist is \"$netlist\""
set netlist [glob -dir $NetlistDirectory *.v]
foreach f $netlist {
	set data $f
	#puts "data is \"$f\""
	puts -nonewline $fileId "\n read_verilog $f"
}
puts -nonewline $fileId "\nhierarchy -check"
close $fileId

```

Whenever we uses **"exec"** command in the tcl script, it runs the command in terminal.

We are runnimg running the yosys tool by passing **"openMSP430.heir.ys"** as input file and catches all the logs in  **"openMSP430.hierarchy_check.log"**

If there is an error, **"my_error"** will be set to 1 and we have to find the error. In yosys when error occurs, then we will find a common pattern such as **"referenced in module"** . It differs across various tools. We have to search each lines of .log file and prints the error statement. The script mentioned below does the above purpose. If the hierarchy check passes then it displays a message saying **"Hierarchy check PASS"**

```
set my_error [catch { exec yosys -s $OutputDirectory/$DesignName.hier.ys >& $OutputDirectory/$DesignName.hierarchy_check.log} msg]
puts "Error flag is \"$my_error\""
if { $my_error } {
	set filename "$OutputDirectory/$DesignName.hierarchy_check.log"
	puts "log file name is \"$filename\" "
	set pattern "referenced in module"
	#puts "pattern is $pattern"
	set count 0
	set fid [open $filename r]
	while {[gets $fid line] != -1} {
		# -- used to say end of command options. everything after this is args
		incr count [regexp -all -- $pattern $line]
		if {[regexp -all -- $pattern $line]} {
			puts "\nError: module [lindex $line 2] is not part of design $DesignName. Please correct RTL in the path '$NetlistDirectory'"
			puts "\nInfo: Hierarchy check FAIL"
		}
	}
	close $fid
} else {
	puts "\nInfo: Hierarchy check PASS"
}
puts "\n Info: Please find the hierearchy check details in [file normalize $OutputDirectory/$DesignName.hierarchy.check.log] for more info"

```

- If an error doesn't occcur, then
<img width="255" height="57" alt="image" src="https://github.com/user-attachments/assets/d7ded2eb-09ff-461a-9c0d-421b103e7fca" />



- If an error occurs during hierarchy check then

<img width="182" height="37" alt="image" src="https://github.com/user-attachments/assets/8b726e72-601e-4b24-b25c-23f0e3bfc097" />




## Module 5: Advanced Scripting Techniques and Quality of Results Generation

In module 5, we will develop script for synthesis and run yosys tool. Then we will talk about procs and discuss some procs that we are going to use in this task. We also convert the foramt [1] and sdc format into format [2] which can be understandable by Opentimer tool.

#### Creating script for Synthesis
```
puts "\nInfo: Creating main synthesis script to be used for yosys"
set data "read_liberty -lib -ignore_miss_dir -setattr blackbox ${LateLibraryPath}"
set filename "$DesignName.ys"
#puts "\nfilename is \"$filename\""
set fileId [open $OutputDirectory/$filename "w"]
#puts "open \"$OutputDirectory/$filename\" in write mode"
puts -nonewline $fileId $data
#puts "netlist is \"$netlist\""
set netlist [glob -dir $NetlistDirectory *.v]
foreach f $netlist {
	set data $f
	#puts "data is \"$f\""
	puts -nonewline $fileId "\n read_verilog $f"
}

puts -nonewline $fileId "\nhierarchy -top $DesignName"
puts -nonewline $fileId "\nsynth -top $DesignName"
puts -nonewline $fileId "\nsplitnets -ports -format __\ndfflibmap -liberty ${LateLibraryPath}\nopt"
puts -nonewline $fileId "\nabc -liberty ${LateLibraryPath}"
puts -nonewline $fileId "\nflatten"
puts -nonewline $fileId "\nclean -purge\niopadmap -outpad BUFX2 A:Y -bits\nopt \nclean"
puts -nonewline $fileId "\nwrite_verilog $OutputDirectory/$DesignName.synth.v"
close $fileId
puts "\nInfo: Synthesis script created and can be accesed from the path $OutputDirectory/$filename"
puts "\nInfo: Running Synthesis...."

if {[catch { exec yosys -s $OutputDirectory/$DesignName.ys >& $OutputDirectory/$DesignName.synthesis.log} msg]} {
	puts "\nError: Syntesis failed due to errors. Please refer to log $OutputDirectory/$DesignName.synthesis.log for errors"
	exit
} else {
	puts "\nInfo: Synthesis finished sucessfully"
}

puts "\nInfo: Please refert olog $OutputDirectory/$DesignName.synthesis.log"

```

- The above script creates a __"openMSP430.ys"__ which can be passed to Yosys for synthesis purpose.
- It writes all the netlist and all scripts necessary for synthesis into __"openMSP430.ys"__
- By using __"exec"__ commnad we can run yosys via tcl command and all the logs are stored in openMSP430.synthesis.log
- If there is an error, it displays a message "Error: Syntesis failed due to errors"


```
set fileId [open /tmp/1 "w"]
puts -nonewline $fileId [exec grep -v -w "*" $OutputDirectory/$DesignName.synth.v]
close $fileId

set output [open $OutputDirectory/$DesignName.final.synth.v "w"]

set filename "/tmp/1"
set fid [open $filename r]
	while {[gets $fid line] != -1} {
	puts -nonewline $output [string map {"\\" ""} $line]
	puts -nonewline $output "\n"
}

close $fid
close $output

puts "\nInfo: Please find the synthesized netlist for $DesignName at below path. You can use this netlist for STA or PNR"
puts "\n$OutputDirectory/$DesignName.final.synth.v"

```

The above scirpt can be used to remove **\** from the gate level netlist. Because opentimer tool cannot understand the netlist with **\**  in it.As we can see we have 6119 **\**  in  the synt.v file as shown in the below figure. After running the above the count came down to 0.

<img width="508" height="742" alt="image" src="https://github.com/user-attachments/assets/58015471-d513-4fbd-88ce-99443f1e8895" />


#### Procs

In Tcl, procedures (commonly called "procs") are a fundamental mechanism for creating reusable script blocks. They function similarly to functions or methods in other programming languages, allowing you to organize script into logical, reusable units.

We are going to use some procs such as reopenStdout.proc, set_num_threads.proc, read_lib.proc, read_verilog.proc, read_sdc.proc to convert the foramt [1] and sdc format to format [2].

1) reopenStdout.proc
 
The __reopenStdout__ proc is a Tcl procedure used to redirect standard output to a file. This technique is sometimes called "reopening" stdout.

```
proc reopenStdout {file} {
    close stdout
    open $file w       
}
```
This procedure works by:

- Closing the current stdout channel

- Opening a file in write mode and making it the new stdout channel

When we call this procedure with a filename as an argument, all subsequent output that would normally go to the console will instead be written to the specified file. This is useful when you need to capture output from commands or procedures that write directly to stdout and don't provide an option to specify an output channel.

2) set_num_threads.proc

The __set_multi_cpu_usage__ proc is a Tcl procedure designed to configure multi-threading capabilities for EDA tools in ASIC design automation workflows. This procedure is part of TCL scripts that automate the frontend of ASIC design.

```
proc set_multi_cpu_usage {args} {
    array set options {-localCpu <num_of_threads> -help "" }
    foreach {switch value} [array get options] {
    puts "Option $switch is $value"
    }
    while {[llength $args]} {
    puts "llength is [llength $args]"
    puts "lindex 0 of \"$args\" is [lindex $args 0]"
        switch -glob -- [lindex $args 0] {
          -localCpu {
              puts "old args is $args"
              set args [lassign $args - options(-localCpu)]
              puts "new args is \"$args\""
              puts "set_num_threads $options(-localCpu)"
              }
          -help {
              puts "old args is $args"
              set args [lassign $args - options(-help) ]
              puts "new args is \"$args\""
              puts "Usage: set_multi_cpu_usage -localCpu <num_of_threads>"
              }
        }
    }
}
```
It accepts command-line style arguments through a parameter array that manages options for local CPU thread allocation. When invoked with the -localCpu flag followed by a numeric value, the procedure configures the underlying EDA tools to utilize the specified number of processor threads for computationally intensive tasks like synthesis and timing analysis. This is accomplished by calling another procedure named set_num_threads with the appropriate thread count. The procedure also includes a helpful -help option that displays usage instructions when requested. 

When __set_multi_cpu_usage -localCpu 8 -help__ commnad is executed, it will go through 2 iterations like in the 1st part shown below. When __set_multi_cpu_usage -localCpu 8__ command is executed, it will go through 1 iterations like shown in the below part.



3) read_verilog.proc
The read_verilog command is used to read Verilog or SystemVerilog source files into design tools. 

```
proc read_verilog {arg1} {
puts "set_verilog_fpath $arg1"
}
```


4) read_lib.proc

The __read_lib proc__ is a Tcl procedure designed to handle library files in ASIC design automation workflows. This procedure is part of a larger TCL scripting framework that automates the frontend of ASIC design.

The procedure accepts multiple options and converts standard library specifications into OpenTimer format.
```
proc read_lib args {
	array set options {-late <late_lib_path> -early <early_lib_path> -help ""}
	while {[llength $args]} {
		switch -glob -- [lindex $args 0] {
		-late {
			set args [lassign $args - options(-late) ]
			puts "set_late_celllib_fpath $options(-late)"
		      }
		-early {
			set args [lassign $args - options(-early) ]
			puts "set_early_celllib_fpath $options(-early)"
		       }
		-help {
			set args [lassign $args - options(-help) ]
			puts "Usage: read_lib -late <late_lib_path> -early <early_lib_path>"
			puts "-late <provide late library path>"
			puts "-early <provide early library path>"
		      }	
		default break
		}
	}
}
```

The procedure has three main options:

- -late: Specifies the path to the late library file used for setup timing analysis

- -early: Specifies the path to the early library file used for hold timing analysis

- -help: Displays usage information for the procedure

When called with the appropriate options, the procedure generates commands in OpenTimer format (e.g., set_late_celllib_fpath and set_early_celllib_fpath) that are used by the timing analysis tool.

This proc works alongside other procedures like read_verilog.proc and read_sdc.proc to convert standard design files into formats compatible with OpenTimer for static timing analysis. It's part of a comprehensive TCL framework for automating ASIC design flows, particularly for timing analysis and quality of results (QoR) generation.

5) read_sdc.proc

__read_sdc__ is a Tcl command used in EDA tools to read Synopsys Design Constraint (SDC) files into the design environment

The read_sdc proc is a large proc file which will be covered in parts.

```
proc read_sdc {arg1} {
set sdc_dirname [file dirname $arg1]
set sdc_filename [lindex [split [file tail $arg1] .] 0 ]
set sdc [open $arg1 r]
set tmp_file [open /tmp/1 "w"] 
puts -nonewline $tmp_file [string map {"\[" "" "\]" " "} [read $sdc]]     
close $tmp_file
}
```
The above script is used to remvoe square bractets "[]" and replace it with "".

```
set tmp_file [open /tmp/1 r]
set timing_file [open /tmp/3 w]
set lines [split [read $tmp_file] "\n"]
set find_clocks [lsearch -all -inline $lines "create_clock*"]
foreach elem $find_clocks {
	set clock_port_name [lindex $elem [expr {[lsearch $elem "get_ports"]+1}]]
	puts "clock_port_name is \"$clock_port_name\" "
	set clock_period [lindex $elem [expr {[lsearch $elem "-period"]+1}]]
	puts "clock_period is \"$clock_period\" "
	set duty_cycle [expr {100 - [expr {[lindex [lindex $elem [expr {[lsearch $elem "-waveform"]+1}]] 1]*100/$clock_period}]}]
	puts "duty_cycle is \"$duty_cycle\" "
	puts $timing_file "clock $clock_port_name $clock_period $duty_cycle"
	}
close $tmp_file

```
The above script is used to convert create_clock constraitns into format [2] which can be understandable by Opentimer tool. Basically it searches a pattern **create_clock** and gets the clock_port_name, clock_period and calucaltes dutry cycle. And then writes all the above values in /tmp/3 file as shown in below figure.

<img width="569" height="131" alt="image" src="https://github.com/user-attachments/assets/f1d0643e-5e07-4c86-8d10-bb64d97ca230" />
<img width="562" height="70" alt="image" src="https://github.com/user-attachments/assets/1a7d6365-ec32-4666-83a5-b6584e933e36" />
<img width="1853" height="953" alt="image" src="https://github.com/user-attachments/assets/b73aa964-222f-427c-9428-0ed69da6f65d" />

```
set find_keyword [lsearch -all -inline $lines "set_clock_latency*"]
set tmp2_file [open /tmp/2 w]
set new_port_name ""
foreach elem $find_keyword {
        set port_name [lindex $elem [expr {[lsearch $elem "get_clocks"]+1}]]
	if {![string match $new_port_name $port_name]} {
        	set new_port_name $port_name
        	set delays_list [lsearch -all -inline $find_keyword [join [list "*" " " $port_name " " "*"] ""]]
        	set delay_value ""
        	foreach new_elem $delays_list {
        		set port_index [lsearch $new_elem "get_clocks"]
        		lappend delay_value [lindex $new_elem [expr {$port_index-1}]]
        	}
		puts -nonewline $tmp2_file "\nat $port_name $delay_value"
	}
}

close $tmp2_file
```

The above script is used to convert set_clock_latency constraitns into format [2] which can be understandable by Opentimer tool. Basically it searches a pattern **set_clock_latency** and gets all the parameters. Then writes all the above values in /tmp/2 file which is furher written in .timings file.
```
set find_keyword [lsearch -all -inline $lines "set_clock_transition*"]
set tmp2_file [open /tmp/2 w]
set new_port_name ""
foreach elem $find_keyword {
        set port_name [lindex $elem [expr {[lsearch $elem "get_clocks"]+1}]]
        if {![string match $new_port_name $port_name]} {
		set new_port_name $port_name
		set delays_list [lsearch -all -inline $find_keyword [join [list "*" " " $port_name " " "*"] ""]]
        	set delay_value ""
        	foreach new_elem $delays_list {
        		set port_index [lsearch $new_elem "get_clocks"]
        		lappend delay_value [lindex $new_elem [expr {$port_index-1}]]
        	}
        	puts -nonewline $tmp2_file "\nslew $port_name $delay_value"
	}
}

close $tmp2_file
set tmp2_file [open /tmp/2 r]
puts -nonewline $timing_file [read $tmp2_file]
close $tmp2_file

```
The above script is used to convert set_clock_transition constraitns into format [2] which can be understandable by Opentimer tool. Basically it searches a pattern **set_clock_transition**  and  gets all the parameters. Then writes all the above values in /tmp/2 file which is furher written in .timings file as shown in below figure.


```
set find_keyword [lsearch -all -inline $lines "set_input_delay*"]
set tmp2_file [open /tmp/2 w]
set new_port_name ""
foreach elem $find_keyword {
        set port_name [lindex $elem [expr {[lsearch $elem "get_ports"]+1}]]
        if {![string match $new_port_name $port_name]} {
                set new_port_name $port_name
        	set delays_list [lsearch -all -inline $find_keyword [join [list "*" " " $port_name " " "*"] ""]]
		set delay_value ""
        	foreach new_elem $delays_list {
        		set port_index [lsearch $new_elem "get_ports"]
        		lappend delay_value [lindex $new_elem [expr {$port_index-1}]]
        	}
        	puts -nonewline $tmp2_file "\nat $port_name $delay_value"
	}
}
close $tmp2_file
set tmp2_file [open /tmp/2 r]
puts -nonewline $timing_file [read $tmp2_file]
close $tmp2_file
```

The above script is used to convert set_input_delay constraitns into format [2] which can be understandable by Opentimer tool. Basically it searches a pattern __set_input_delay__ and gets all the parameters. Then writes all the above values in /tmp/2 file which is furher written in .timings file 


```
set find_keyword [lsearch -all -inline $lines "set_input_transition*"]
set tmp2_file [open /tmp/2 w]
set new_port_name ""
foreach elem $find_keyword {
        set port_name [lindex $elem [expr {[lsearch $elem "get_ports"]+1}]]
        if {![string match $new_port_name $port_name]} {
                set new_port_name $port_name
        	set delays_list [lsearch -all -inline $find_keyword [join [list "*" " " $port_name " " "*"] ""]]
        	set delay_value ""
        	foreach new_elem $delays_list {
        		set port_index [lsearch $new_elem "get_ports"]
        		lappend delay_value [lindex $new_elem [expr {$port_index-1}]]
        	}
        	puts -nonewline $tmp2_file "\nslew $port_name $delay_value"
	}
}

close $tmp2_file
set tmp2_file [open /tmp/2 r]
puts -nonewline $timing_file [read $tmp2_file]
close $tmp2_file

```
The above script is used to convert set_input_transition constraitns into format [2] which can be understandable by Opentimer tool. Basically it searches a pattern __set_input_transition__  and then gets all the parameters and convert into Opentimer format. Then writes all the above values in /tmp/2 file which is furher written in .timings file.

```
set find_keyword [lsearch -all -inline $lines "set_output_delay*"]
set tmp2_file [open /tmp/2 w]
set new_port_name ""
foreach elem $find_keyword {
        set port_name [lindex $elem [expr {[lsearch $elem "get_ports"]+1}]]
        if {![string match $new_port_name $port_name]} {
                set new_port_name $port_name
        	set delays_list [lsearch -all -inline $find_keyword [join [list "*" " " $port_name " " "*"] ""]]
        	set delay_value ""
        	foreach new_elem $delays_list {
        		set port_index [lsearch $new_elem "get_ports"]
        		lappend delay_value [lindex $new_elem [expr {$port_index-1}]]
        	}
        	puts -nonewline $tmp2_file "\nrat $port_name $delay_value"
	}
}

close $tmp2_file
set tmp2_file [open /tmp/2 r]
puts -nonewline $timing_file [read $tmp2_file]
close $tmp2_file
```

The above script is used to convert set_output_delay constraitns into format [2] which can be understandable by Opentimer tool. Basically it searches a pattern __set_output_delay__  and then gets all the parameters and convert into Opentimer format. Then writes all the above values in /tmp/2 file which is furher written in .timings file.

```
set ot_timing_file [open $sdc_dirname/$sdc_filename.timing w]
set timing_file [open /tmp/3 r]
while {[gets $timing_file line] != -1} {
        if {[regexp -all -- {\*} $line]} {
                set bussed [lindex [lindex [split $line "*"] 0] 1]
                set final_synth_netlist [open $sdc_dirname/$sdc_filename.final.synth.v r]
                while {[gets $final_synth_netlist line2] != -1 } {
                        if {[regexp -all -- $bussed $line2] && [regexp -all -- {input} $line2] && ![string match "" $line]} {
                        puts -nonewline $ot_timing_file "\n[lindex [lindex [split $line "*"] 0 ] 0 ] [lindex [lindex [split $line2 ";"] 0 ] 1 ] [lindex [split $line "*"] 1 ]"
                        } elseif {[regexp -all -- $bussed $line2] && [regexp -all -- {output} $line2] && ![string match "" $line]} {
                        puts -nonewline $ot_timing_file "\n[lindex [lindex [split $line "*"] 0 ] 0 ] [lindex [lindex [split $line2 ";"] 0 ] 1 ] [lindex [split $line "*"] 1 ]"
                        }
                }
        } else {
        puts -nonewline $ot_timing_file  "\n$line"
        }
}

close $timing_file
puts "set_timing_fpath $sdc_dirname/$sdc_filename.timing"
}

```
The above script is used to expand bussed ports into format [2] which can be understandable by Opentimer tool as shown below.


#### Creating scripts for Opentimer

```

if {$enable_prelayout_timing == 1} {
	puts "\nInfo: enable prelayout_timing is $enable_prelayout_timing. Enabling zero-wire load parasitics"
	set spef_file [open $OutputDirectory/$DesignName.spef w]
	puts $spef_file "*SPEF \"IEEE 1481-1998\""
	puts $spef_file "*DESIGN \"$DesignName\""
	puts $spef_file "*DATE \"Sun Jun 11 11:59:00 2023\""
	puts $spef_file "*VENDOR \"VLSI System Design\""
	puts $spef_file "*PROGRAM \"TCL Workshop\""
	puts $spef_file "*DATE \"0.0\""
	puts $spef_file "*DESIGN FLOW \"NETLIST_TYPE_VERILOG\""
	puts $spef_file "*DIVIDER /"
	puts $spef_file "*DELIMITER : "
	puts $spef_file "*BUS_DELIMITER [ ]"
	puts $spef_file "*T_UNIT 1 PS"
	puts $spef_file "*C_UNIT 1 FF"
	puts $spef_file "*R_UNIT 1 KOHM"
	puts $spef_file "*L_UNIT 1 UH"
}
close $spef_file
```
The above script creates a .spef file and writes all the required commands into it as shown in below



```
set conf_file [open $OutputDirectory/$DesignName.conf a]
puts $conf_file "set_spef_fpath $OutputDirectory/$DesignName.spef"
puts $conf_file "init_timer"
puts $conf_file "report_timer"
puts $conf_file "report_wns"
puts $conf_file "report_tns"
puts $conf_file "report_worst_paths -numPaths 10000 " 
close $conf_file
```

The above script creates a .conf file and writes all the required commands into it which is then passed as input to Opentimer tool as shown in below



#### Quality of results (QOR) generation algorithm

This is the final sub-task which involves output generation as a datasheet.

```
set time_elapsed_in_us [time {exec /home/vsduser/OpenTimer-1.0.5/bin/OpenTimer < $OutputDirectory/$DesignName.conf >& $OutputDirectory/$DesignName.results} ]
set time_elapsed_in_sec "[expr {[lindex $time_elapsed_in_us 0]/100000}]sec"
puts "\nInfo: STA finished in $time_elapsed_in_sec seconds"
puts "\nInfo: Refer to $OutputDirectory/$DesignName.results for warings and errors"

```
The above script is used to executed the Opnetimer tool by passing openMSP430.conf file as an input the results are stored in openMSP430.results. It also stores the time elapsed during STA in microseconds and seconds as shown below.

<img width="698" height="125" alt="image" src="https://github.com/user-attachments/assets/79d4b047-b95f-454a-8b88-070ec550662f" />




```
#-------------------- Finding worst outptut violation --------------------------#
set worst_RAT_slack "-"
set report_file [open $OutputDirectory/$DesignName.results r]
set pattern {RAT}
while {[gets $report_file line] != -1} {
	if {[regexp $pattern $line]} {
		set worst_RAT_slack "[expr {[lindex $line 3]/1000}]ns"
		break
	} else {
		continue
	}
}
close $report_file
#--------------------------- Finding number of outptut violation ------------------------#
set report_file [open $OutputDirectory/$DesignName.results r]
set count 0
while {[gets $report_file line] != -1} {
	incr count [regexp -all -- $pattern $line]
}
set Number_output_violations $count
close $report_file

#--------------------------- Finding worst setup violation -------------------------#
set worst_negative_setup_slack "-"
set report_file [open $OutputDirectory/$DesignName.results r] 
set pattern {Setup}
while {[gets $report_file line] != -1} {
	if {[regexp $pattern $line]} {
		set worst_negative_setup_slack "[expr {[lindex $line 3]/1000}]ns"
		break
	} else {
		continue
	}
}
close $report_file


#-------------------------find number of setup violations--------------------------------#
set report_file [open $OutputDirectory/$DesignName.results r]
set count 0
while {[gets $report_file line] != -1} {
	incr count [regexp -all -- $pattern $line]
}
set Number_of_setup_violations $count
close $report_file

#-------------------------find worst hold violation--------------------------------#
set worst_negative_hold_slack "-"
set report_file [open $OutputDirectory/$DesignName.results r] 
set pattern {Hold}
while {[gets $report_file line] != -1} {
	if {[regexp $pattern $line]} {
		set worst_negative_hold_slack "[expr {[lindex $line 3]/1000}]ns"
		break
	} else {
		continue
	}
}
close $report_file

#-------------------------find number of hold violations--------------------------------#
set report_file [open $OutputDirectory/$DesignName.results r]
set count 0
while {[gets $report_file line] != -1} {
	incr count [regexp -all -- $pattern $line]
}
set Number_of_hold_violations $count
close $report_file

#-------------------------find number of instances--------------------------------#

set pattern {Num of gates}
set report_file [open $OutputDirectory/$DesignName.results r] 
while {[gets $report_file line] != -1} {
	if {[regexp $pattern $line]} {
		set Instance_count "[lindex [join $line " "] 4 ]"
		break
	} else {
		continue
	}
}
close $report_file


puts "DesignName is \{$DesignName\}"
puts "time_elapsed_in_sec is \{$time_elapsed_in_sec\}"
puts "Instance_count is \{$Instance_count\}"
puts "worst_negative_setup_slack is \{$worst_negative_setup_slack\}"
puts "Number_of_setup_violations is \{$Number_of_setup_violations\}"
puts "worst_negative_hold_slack is \{$worst_negative_hold_slack\}"
puts "Number_of_hold_violations is \{$Number_of_hold_violations\}"
puts "worst_RAT_slack is \{$worst_RAT_slack\}"
puts "Number_output_violations is \{$Number_output_violations\}"
```

The above script can be used to find different parameters such as worst hold violation, no of hold vioalations etc... which are reported in final output and also displays on terminal.

<img width="502" height="205" alt="image" src="https://github.com/user-attachments/assets/d30e6cde-70e1-4e95-9323-d6c470dca3ce" />


```
puts "\n"
puts "						****PRELAYOUT TIMING RESULTS**** 					"
set formatStr "%15s %15s %15s %15s %15s %15s %15s %15s %15s"

puts [format $formatStr "----------" "-------" "--------------" "---------" "---------" "--------" "--------" "-------" "-------"]
puts [format $formatStr "DesignName" "Runtime" "Instance Count" "WNS Setup" "FEP Setup" "WNS Hold" "FEP Hold" "WNS RAT" "FEP RAT"]
puts [format $formatStr "----------" "-------" "--------------" "---------" "---------" "--------" "--------" "-------" "-------"]
foreach design_name $DesignName runtime $time_elapsed_in_sec instance_count $Instance_count wns_setup $worst_negative_setup_slack fep_setup $Number_of_setup_violations wns_hold $worst_negative_hold_slack fep_hold $Number_of_hold_violations wns_rat $worst_RAT_slack fep_rat $Number_output_violations {
	puts [format $formatStr $design_name $runtime $instance_count $wns_setup $fep_setup $wns_hold $fep_hold $wns_rat $fep_rat]
}

puts [format $formatStr "----------" "-------" "--------------" "---------" "---------" "--------" "--------" "-------" "-------"]
puts "\n"
	
```
The above script can be used to show the results in Horizontal format like shown in below figure.

![Image](https://github.com/user-attachments/assets/bfb816b1-82a4-4902-9227-75c82088b225)
</div>
