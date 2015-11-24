#!/bin/bash
#This program converts drill files to G-code files based on several parameters

if [[ ! $# =~ [89] ]]; then
	echo "Correct usage: drl2ngc <drill file> <diameter of endmill> <lateral feed rate> <vertical feedrate> <rapid movement speed> <rapid movement height> <drill depth> <output file> [fast]
--------
Note that all units are in inches or inches/minute, and absolute; ie drill depth is Z value to drill down to)
--------
Appending 'fast' to the command will make the script execute faster, however results in a less efficient toolpath, as you can see by comparing the estimated completion times. The fast option is useful for quickly optimizing your parameters and observing the corresponding change in estimated completion time, to achieve an even more efficient toolpath
--------
Example usage: ./drl2ngc holes.drl .025 8 1 60 0.05 -0.1 holes.ngc fast"
	exit 1
fi

#nomenclature definition mapping
inputFile=$1
endmillDiameter=$2
lateralFeedRate=$3
verticalFeedRate=$4
rapidMovementSpeed=$5
rapidMovementHeight=$6
drillDepth=$7
outputFile=$8
mode=$9

#initializing some general variables
totalLines=$(cat $inputFile | wc -l) #total lines in input file
currentLine=1 #pointer to current line of input file
toolDescriptionSection=-1 #pointer to the beginning of the section of the input file that describes tool widths
currentTool=1 #keep track of which tool (hole size) we are working with
workSection=-1 #pointer to the beginning of the section of the input file that describes the holes to drill
numberOfHoles=0 #total number of holes to drill
warnings="" #output warning messages after completion of the file

#Some setup stuff: analyzing input file, preparing variables, etc.

#warn if parameters are wonky
if [[ 1 = $(echo "$rapidMovementHeight < $drillDepth" | bc) ]]; then
	warnings+="Rapid movement height, $rapidMovementHeight is below drill depth, $drillDepth\n"
fi
if [[ 1 = $(echo "$rapidMovementSpeed < $lateralFeedRate" | bc) ]]; then
	warnings+="Rapid movement speed, $rapidMovementSpeed is slower than milling speed, $lateralFeedRate\n"
fi
if [[ 0 = $(echo "$endmillDiameter > 0" | bc) ]]; then
	warnings+="Endmill diameter, $endmillDiameter should be greater than zero\n"
fi
if [[ 0 = $(echo "$lateralFeedRate > 0" | bc) ]]; then
	warnings+="Lateral feed rate, $lateralFeedRate should be greater than zero\n"
fi
if [[ 0 = $(echo "$verticalFeedRate > 0" | bc) ]]; then
	warnings+="Vertical feed rate, $verticalFeedRate should be greater than zero\n"
fi
if [[ 0 = $(echo "$rapidMovementSpeed > 0" | bc) ]]; then
	warnings+="Rapid movement speed, $rapidMovementSpeed should be greater than zero\n"
fi

#find tool description section, number of tools, work section, and number of holes.
#construct an array, from which we can determine what holes have yet to be drilled, and how close they are to our current position
while [ $currentLine -le $totalLines ]; do
	sedArg="$currentLine$()q;d"
	if [[ $(sed $sedArg $inputFile) =~ T[0-9]+C[0-9]+ ]]; then
		
		#put the diameter of tool into an array	
		toolNumber=$(echo $(sed $sedArg $inputFile | sed -e 's/[tT]//') | sed -e 's/[cC].*$//')	
		toolDiameter[$toolNumber]=$(sed $sedArg $inputFile | sed -e 's/[tT][0-9]\+[cC]//g')
		tmpOffset=${toolDiameter[$toolNumber]}
		toolOffset[$toolNumber]=$(echo "scale=4; ($tmpOffset - $endmillDiameter)/2" | bc)

		#warn if endmill is larger than hole
		if [[ 1 = $(echo "$endmillDiameter >= ${toolDiameter[$toolNumber]}" | bc) ]]; then
			echo -e "\nTool $toolNumber, with a diameter of ${toolDiameter[$toolNumber]} is smaller than or equal to the specified endmill, with a diameter of $endmillDiameter\n"
			exit 1			
		fi

		((currentLine+=1))
		continue
	fi

	if [[ $(sed $sedArg $inputFile) =~ ^[tT]1$ ]]; then
		workSection=$currentLine
		((currentLine+=1))
		continue
	fi

	if [[ $(sed $sedArg $inputFile) =~ ^[tT][0-9]+$ ]]; then
		currentTool=$(sed $sedArg $inputFile | sed -e 's/[tT]//')
		((currentLine+=1))
		continue
	fi

	if [[ $(sed $sedArg $inputFile) =~ ^[xX][0-9] ]]; then
		xArray[$numberOfHoles]=$(echo $(echo $(sed $sedArg $inputFile) | sed -e 's/^[xX]//') | sed -e 's/[yY].*//')
		yArray[$numberOfHoles]=$(echo $(sed $sedArg $inputFile) | sed -e 's/^[xX].*[yY]//')
		toolArray[$numberOfHoles]=$currentTool
		toBeDrilled[$numberOfHoles]="true"
		((numberOfHoles+=1))
		((currentLine+=1))
		continue
	fi
	((currentLine+=1))
done

echo $numberOfHoles holes

#prep the output file
echo > $outputFile
echo -e "G20 ;set to inches mode, as opposed to mm" >> $outputFile
echo -e "G90 ;absolute distance mode" >> $outputFile
echo -e "G17 ;set helix axis of rotation to Z axis" >> $outputFile
echo -e "G40 ;Cancel cutter radius compensation " >> $outputFile
echo -e "G49 ;Cancel tool length offset" >> $outputFile
echo -e "G80 ;Cancel motion mode" >> $outputFile
echo -e "\n;raise tool to safety height\nF $verticalFeedRate\nG0 Z$rapidMovementHeight\nF $rapidMovementSpeed" >> $outputFile

#main loop that converts holes to g-code, considering tool diameters/offsets
#also keeps track of estimated time it will take to mill the object

millTime=0 #estimated completion time
oldx=0
oldy=0
thisHole="null"
holesComplete=0

while [ true ]; do
	if [[ $mode ]]; then
		thisHole=$holesComplete
	else
		#Find next nearest hole
		shortestDistance="null"
		closestHole="null"
		counter=0
		while [[ $counter -lt $numberOfHoles ]]; do
			if [ ${toBeDrilled[$counter]} = "true" ]; then
				tmpx=${xArray[$counter]}
				tmpy=${yArray[$counter]}
				tmpOffset=${toolOffset[${toolArray[$counter]}]}
				#tmpy=$(echo "scale=4; $tmpy + $tmpOffset" | bc)
				distance=$(echo "scale=4; (($oldx - $tmpx)^2 + ($oldy - $tmpy)^2)" | bc)
				if [[ $shortestDistance = "null" || 1 = $(echo "$distance < $shortestDistance" | bc) ]]; then
					shortestDistance=$distance
					closestHole=$counter
				fi
			fi
			((counter+=1))		
		done
		thisHole=$closestHole
		toBeDrilled[$thisHole]="false"
	fi
	
	
	#set new tool and offset	
	currentTool=${toolArray[$thisHole]}
	offset=${toolOffset[$currentTool]}
	
	#set x and y to hole center, while yOffset is the actual y value to move to, accounting for diameter of hole and endmill
	x=${xArray[$thisHole]}
	y=${yArray[$thisHole]}
	yOffset=$(echo "scale=4; $y+$offset" | bc)
	
	#move above hole and set lateral feed rate to helix down
	echo -e "\nG1 X$x Y$yOffset\nG1 Z0\nF $lateralFeedRate" >> $outputFile

	#spiral down to drill depth, first by calculating the required rotations to match lateral and vertical feedrates
	#vertical speed rate will be equal to, or marginally slower that the set vertical feed rate
	pValue=$(echo "scale=6; ($rapidMovementHeight - $drillDepth ) / ( $verticalFeedRate * ( 3.14159 * 2 * $offset / $lateralFeedRate ))" | bc)
	pValue=$(echo "$pValue / 1" | bc)
	counter=1
	while [[ $counter -le $pValue ]]; do
		tmpZ=$(echo "scale=4; $drillDepth / $pValue * $counter" | bc)
		echo -e "G2 X$x Y$yOffset Z$tmpZ I0 J-$offset" >> $outputFile
		((counter+=1))
	done
	#make a full circle at drill depth to guarantee proper drilling
	echo -e "G2 X$x Y$yOffset Z$drillDepth I0 J-$offset" >> $outputFile

	#return to rapid movement height and speed
	echo -e "F $rapidMovementSpeed\nG1 Z$rapidMovementHeight" >> $outputFile
	
	#calculate time this operation will take and add it to total mill time
	millTime=$(echo "scale=4; $millTime + sqrt((($x - $oldx)^2+($yOffset - $oldy)^2))/$rapidMovementSpeed" | bc)	
	millTime=$(echo "scale=4; $millTime + ($pValue + 1)*(3.14159 * 2 * $offset)/($lateralFeedRate)" | bc)
	millTime=$(echo "scale=4; $millTime + ($rapidMovementHeight - $drillDepth)/$rapidMovementSpeed" | bc)	
	oldx=$x
	oldy=$yOffset

	((holesComplete+=1))		
	
	#print percentage complete
	percent=$(echo "scale=4; 100 * $holesComplete / $numberOfHoles" | bc)
	percent=$(echo "$percent / 1" | bc)
	echo -ne "\rProgress: ["
	counter=1
	while [[ $counter -le 20 ]]; do
		if [[ $(($counter*5)) -le $percent ]]; then
			echo -n '#'
		else
			echo -n '.'
		fi
		((counter+=1))
	done
	echo -n "]($percent%)"

	if [[ $holesComplete = $numberOfHoles ]]; then
		break
	fi
	
done

echo -e "\nM5 ;Stop spindle\nM2 ;End program" >> $outputFile

echo -e "\n-----\nDone\nCalculated completion time: $millTime minutes\n"

if [[ $warnings != "" ]]; then
	echo -----------Warnings!------------
	echo -e $warnings
fi
