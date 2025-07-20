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
   Case 2: Non-existent file
   ```
   ./vsdsynth dummy.csv
   ```
   Case 3: Help
   ```
   ./vsdsynth -help
   ```
   Case 4: Valid CSV file
   Create a dummy CSV:
   ```
   touch test.csv
   ```
   Then run:
   ```
   ./vsdsynth test.csv
   ```

   
