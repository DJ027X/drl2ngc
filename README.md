# drl2ngc
Bash script to convert Excellon drill files (.drl) to g-code (.ngc) files, so you can use a grbl (or equivalent) controlled endmill to drill any number of arbitrarily sized hole

Note that this is a linux script, however it can be run on windows via cygwin, provided the 'bc' package is installed

Correct usage: drl2ngc \<drill file\> \<diameter of endmill\> \<lateral feed rate\> \<vertical feedrate\> \<rapid movement speed\> \<rapid movement height\> \<drill depth\> \<output file\> [fast]
Note that all units are in inches or inches/minute, and absolute; ie drill depth is Z value to drill down to)
Appending 'fast' to the command will make the script execute faster, however results in a less efficient toolpath, as you can see by comparing the estimated completion times. The fast option is useful for quickly optimizing your parameters and observing the corresponding change in estimated completion time, to achieve an even more efficient toolpath
Example usage: ./drl2ngc holes.drl .025 8 1 60 0.05 -0.1 holes.ngc fast

This bash script will take an Excellon formatted drill file and convert it into a toolpath described in g-code that will mill out the holes described in the Excellon file over a flat plane, and give you a very accurate estimate of how long it will take your CNC machine to complete the job. The script takes into account the diameter of the endmill, so as to inset the endmill when carving the hole to ensure that the diameter of the carved hole is correct. This allows you to use any size endmill to carve the holes, as long as the hole is larger than the endmill. Instead of plunging the endmill, which is very undesireable behaviour, the script generates a helical ramp that spirals downward to a set cutting depth, makes one more full revolution to ensure all necessary material is removed, and raises back up to a set safety height. Several checks are put in place to warn the user if the parameters they entered conflict with each other. You may define a particular lateral, and vertical feed rate, as well as rapid movement speed, which the generated g-code will closely adhere to. While most contingencies have been considered, it may still be possible to damage your machine by running the generated g-code. For this reason, I recommend you use a simulation program such as CAMotics, as well as doing a dry-run with the CNC mill before doing a production run. Use this software at your own risk.

Feel free to suggest additional features, and I will consider appedning them to the script.
