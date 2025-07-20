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
7. Find Start Indexes of Clock/Input/Output Constraints
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
8. Parse and Convert to SDC Format (To be Added in Step 9)
9. Collect All Verilog Files and Write Yosys Script
   ```
   set verilog_files [glob -nocomplain -directory $NetlistDirectory *.v]
   set yosys_script "$OutputDirectory/synth.ys"
   set ys [open $yosys_script "w"]

   puts $ys "read_liberty -lib $EarlyLibraryPath"
   foreach file $verilog_files {
       puts $ys "read_verilog $file"
   }
   puts $ys "synth -top $DesignName"
   puts $ys "write_verilog $OutputDirectory/${DesignName}_synth.v"
   puts $ys "write_sdc $OutputDirectory/${DesignName}_synth.sdc"

   close $ys
   ```
10. Run Yosys Synthesis
    ```
    exec yosys $yosys_script
    ```
    


