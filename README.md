# MAPFScenario

## What is it
Mapf Scenario holds for Multi Agent Path Finding Scenario. It is program that can:
- work with grid maps. Allows to design, print, export as image.
- specify Multiple Agent Path Finding problem. (place on grid map agents start position and agents target positions)
- using picat interpreter and provided solvers to find solution to MAPF problem.
- visualize solution directly on drawn map
- export MAPF solution as ozobot program.  

## How to run it (UBUNTU)
 MAPF Scenario uses graphic library javafx which can be installed by typing
 ```
 sudo apt-get install openjfx
 ```
to create jar archive is used apache ant build, which can be installed by  
```
sudo apt-get install ant
 ```
It may be needed to check if libraries are located under right JAVA_HOME path.
To check that open build.xml find property 	 
```<property name="JAVA_HOME" value="/usr/share/java"/>```
if under given folder aren't present javafx jar packages it may be needed to locate right path. 

```
ant publish
```
should publish project. 

to run project just type (from publish directory)
```
java -jar MAPFScenario.jar
```
Happy developing. 

# Developer Notes:  

## File structure
- MapfScenario.jar - main executable. 
- config.properties - used to store program configuration
- action_durations - used in simulation. contains custom durations of the ozobot actions.
- template.ozocode - ozocode program file which is used by Ozocode generator as template to produce pathfinding program.
- workdir - program working direcory. Contains files "pr<current_date>.pi" which are used by picatWrapper.jar, and "out<current_date>.answ" which contains ouptput of picatWrapper.jar, and are used by simulator and ozocode generator.    
- picat/picat_iface.pi - picat interface is file which contains predicates that picat exposes and can be used in MapfScenario. More about picat can be found on http://picat-lang.org/ .
- lib/OzocodeGenerator.jar - runnable archive which read files: "out<date>.answ" template.ozocode. and produces ozocode made of out<date>.answ file. 
- lib/PicatWrapper.jar - runnable archive used to call platform dependent picat interpreters. Currently supports only linux and macOs.    

 
## Program can be divided into following logical components:
- OzoCodeGenerator. Used to generate ozobot code. It is separate executable jar archive.
- PicatWrapper. Platform independent wrapper over picat libraries. Used to call picat methods from java. 
  picat folder - contains picat scripts, which are used to model and find solution for generated problem.  
- workdir - folder where are files that are used to exchange information beetwen components.  
- MapfScenario - gui program which is used interactively create multi agent path finding problem, call solver and vizualize result. (It uses internally OzocodeGenerator and PicatWrapper)


## On button solver clicked workflow
1. currently drawn map is converted to graph, and with agents positions is written into file "workdir/pr<current_date>.pi"
2. picat interpreter (picatWrapper) is invoked with "pr<current_date>.pi" file path and file where "out<date>.answ" where picat output is expected
3. after picat interpreter had finished. "out<date>.answ" is read and parsed.  


## Workdir file structures
Workdir is folder used to store files whose are subject of passing to another component.

### pr&lt;date>.pi
This file is generated when in MapfScenario button Solve is clicked. It contains all data about currently modeled problem written in picat term format.  

File consists of graph of nodes, and agent start and end positions
ins(IncidentGraph,AgentsPositions).

IncidentGraph = [neibs(VertexId,ListOfNeigbours), ... ]   
AgentPositions = [ (AgentVertexIdStart,AgentVertexIdEnd),... ]

IncidentGraph is a list of terms neibs, where each term is representing graph vertexId and contains list of its neigbours.
AgentPositions is a list of tuples where each tuple represnets one agent, and first value is start position of agent, second is agent goal position. 

file also contains term idPosMap which is map that can be used to map vertexIds back to grid location. 
```
Example file:
% Map num mapping
%(0,2) => 5
%(2,0) => 1
%(2,2) => 6
%(4,0) => 2
%(6,0) => 3
%(6,2) => 7
%(8,0) => 4
%(8,2) => 8

ins([
neibs(1,[2,6]),
neibs(2,[1,3]),
neibs(3,[2,4,7]),
neibs(4,[3,8]),
neibs(5,[6]),
neibs(6,[1,5]),
neibs(7,[3,8]),
neibs(8,[4,7])
],[(5,4),(8,5)]).
% Map from ID to coords (x, y) top left is (0,0) 
idPosMap([1 =(2,0),2 =(4,0),3 =(6,0),4 =(8,0),5 =(0,2),6 =(2,2),7 =(6,2),8 =(8,2)]).
```

Represents following map: format: nodeId(xCord,yCord)
```
         1(2,0)--2(4,0)--3(6,0)--4(8,0) 
           |               |       |
 5(0,2)--6(2,2)          7(6,2)--8(8,2)
```
Where there are two agents. One starts on node 5(bottom left), and wants to go to node 4 (top right), and second is on node 8 and want to go to node 5.  


### out&lt;date>.answ

This file is generated by picatWrapper after button Solver in MapfScenario is clicked.
To be more precise: MapfScenario generates file "pr<data>.pi" and gives it as argument to picatWrapper, which reads file and writes upon it solution file ("out<date>.answ"). This solution file is further used to visualize solution and to generate ozobot program. 

This file is readed line by line and contains picat generated plan for agents.
One line represents agent current position. Combination of two neigbour lines gives agent action. 

Example:
```
1 3 null east 2000 "east" 
1 4 null end 2000 "end"  
```
This says, that agent number 1, starts on vertex 3. performs operation east, which takes 2000 miliseconds. And after that operation agent is on vertex 4.

File contains six columns which are divided by space. 

Coumnus description:
1. AgentNo - numbered from 1. Defines for which agent action holds.
2. VertexNo - vertexId as used in "pr<date>.pi". Specifies Agent position beetween action.  
3. Rotation - can be ommited and set by null. It is clockwise value of agent current rotation in degrees. Zero is top, supports only nonnegative integer values
4. Action - it is label which says what action will be performed between current line and next one. Last action of each agent should be always "end".
5. Duration - if picat script supports action durations (miliseconds). It can specify how long each action will take. From view point of this file, action is transition beetween current line and the next one.
6. SomeText - for any debugging or another purposese. On simulation this text should be displayed when mouse hovered over its action  


## Picat
Picat language and runtime is used to model Multiagent pathfinding algorithms. 
In folder picat is located file picat_iface.pi. When MapfScenario starts it reads this file line by line
and searches for lines starting with "%[SCENARIO_EXPORT]". For example:

```
%[SCENARIO_EXPORT] mapf_edge_split EdgeSplit
```

This line indicates that there is valid picat prediate mapf_edge_split that can be called from picat_iface.pi.
EdgeSplit is alias to given predicate which will be included in MapfScenario solver list.
  
MapfScenario expects that given predicate will contain two arguments: Input_File, Output_File.
Input_File is path to existing file formated as pr<date>.pi.
Output_file is path to non existing file, to where given predicate should write algorithms result formatted as out<date>.answ.  

## Own Picat Solver (Method A)
It is possible to write own picat solver altorithm. To do that it is needed to write unique predicate in picat_iface.pi with two arguments where from first it reads input file in format "pr<date>.pi" and processed solution writes to file given by second argument, with format "out<date>.answ".
Note that given given predicate should be labelled with comment 

```
%[SCENARIO_EXPORT] newly_writen_predicate aliasOfNewlyWrittenPredicateInMapfScenario
```
which is used to index newly created predicate. 


## Own Picat Solver (Method B)
It is also possible to specify custom solver algorithm inside MapftScenario by selecting solver file. Then it is expected that this file will contain predicate (default "run") specified in Settings-> "predicate from custom file". 
Also that predicate needs to have two arguments: first that contains path to file "pr<date>.pi" and second for file where will be written solver output in format of file "out<date>.answ".


## PicatWrapper jar archive
Because picat interperter is platform dependent. It was compiled for linxu and macOs as a library.
And over it was written a simple wrapper, which calls picat interpreter with given picat file and predicate.
   
Aruments to call picatWraper manually:
workdir - folder where picat script is.
called Predicate - predicate with argumetns that will be run
picatModule - picat file with predicate to run. 

Example: ./picat mapf_edge_split("Full_Path.pi","Full_path.answ") picat_iface.pi



## Ozocode generator (jar archive)
Main purpose of this module is to convert picat solution file "out<date>.answ" to loadable ozobot program. 

Ozobot program is a xml file.  To avoid future possible language changes. Creating program is done by xml inection into existing template.ozocode file. 

Suggested approach is following:
- In ozoblockly create progam which contains needed functions named as actions that are used in picat i.e. ones that can be seen in simulation. Body of these function is the code that ozobot has to perform for given action. And should contain corresponding code to that action. 
- Create mock function named "ENTRYPOINT" (without quotes) and add a single call of this function. No other executable code sould be present. (except function definitions)
- Export this program to file for example template.ozocode

(A) When ozocode is generated from Mapf scenario:
- Specify path to template from tab settings-> ozobot template file.    
- click ozo export on showed solution

(B) When ozocode is genereted manually from OzocodeGenerator.jar:

- run command :
```
  java -jar OzocodeGenerator.jar <solution_input_file> <template_ozoblockly_xml> <output_ozobolockly_xml> agentNumber
```  
   where
   <solution_input_file> is file "out<date>.answ"
   <template_ozoblockly_xml> is prevously generated ozobot code
   <output_ozoblockly_xml>  is name of outpu file where template with injected code will be
   agentNumer is specified for what agent should be program generated. 

Note: Result ozocode program is a copy of generated template.ozocode, where call of function "ENTRYPOINT" is substituted with sequence of function calls which are actions given by solver algorthm.
  
    

