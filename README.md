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
}
   ```
