# TCL_Workshop
TCL (Tool Command Language) is a versatile scripting language commonly used to automate processes in VLSI design. From design setup to verification, TCL helps streamline workflows, making it a key part of modern EDA tools and custom automation.

## Day 1: Introduction to TCL and VDSYNTH Toolbox Usage
In this workshop, we are building a TCL script (referred to as a TCL box), which takes a CSV file as input and generates timing results as output. 

<img width="1424" height="727" alt="image" src="https://github.com/user-attachments/assets/030d4998-ed86-4216-82b0-13ad32359ac0" />

The given task is further divided into subtasks - 
1. Create a custom command (ex. vsdsynth) to pass .csv files from the UNIX shell to the TCL script
2. Convert .csv input to format [1] (as required by the TCL script) and SDC (Synopsys Design Constraints) format
3. The converted inputs are passed to Yosys synthesis tool (conversion to format [1] is done as Yosys does not support .csv files directly)

