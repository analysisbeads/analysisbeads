Get the tools and the classical games including some engine from this zip file:  
https://github.com/analysisbeads/analysisbeads/releases/download/v0.69/analysisbeads.zip

If you just want to play with the final data (ACPL and CPL for the 14 players analysed):  
https://github.com/analysisbeads/analysisbeads/releases/download/v0.69/csv.zip

The compiled thisweekinchess.com archive:  
https://github.com/analysisbeads/analysisbeads/releases/download/v0.69/2012-2019.pgn.zip  
https://github.com/analysisbeads/analysisbeads/releases/download/v0.69/2019-2022.pgn.zip


### Analysisbeads

Instructions how to extract and analyse chess games in PGN format on Debian Linux.

### Tools Needed:

[pgn-extract](https://www.cs.kent.ac.uk/people/staff/djb/pgn-extract/)  to work with .pgn files

[uci-analyser](https://www.cs.kent.ac.uk/people/staff/djb/uci-analyser/) to analyse .pgn files

unzip

python3

ripgrep (optional utility)

pgn-extract and uci-analyser are included in the release zip, the others can be installed with:

`apt install ripgrep unzip python3`


### 1. Data Aquisition

You can use any .pgn files. I downloaded the entire archive from https://theweekinchess.com/twic  
This archive is included in the release package, so you don't have to hammer their servers.

### 2. Data Preparation

Extract the [release archive](https://github.com/analysisbeads/analysisbeads/releases/download/v0.69/analysisbeads.zip) and navigate into the extracted dir in a terminal window.  
For example on my machine thats:  
`cd /home/centipawn/chess/`

If you're in the right dir `ls` should show you these dirs:  
engines games results results_sf11 temp tools

The games directory contains several .pgn files you can extract the ones you're want from.  
I worked with [2019-2022.pgn](https://github.com/analysisbeads/analysisbeads/releases/download/v0.69/2019-2022.pgn.zip) for all players except Rausis, I used [2012-2019](https://github.com/analysisbeads/analysisbeads/releases/download/v0.69/2012-2019.pgn.zip) for him to get more games.

**First step is to separate the games into offline and online based on "Site" tag of the pgn (INT = online)**  
`tools/pgn-extract -t "tools/filters/online.txt" -n "games/offline.pgn" -o "games/online.pgn" "games/2019-2022.pgn"`

games/offline.pgn is the file that contains all offline games after extraction  
games/online.pgn is the file that contains all online games after extraction  
games/2019-2022.pgn is the input file that you want to extract the games from  

*this has already been done for you if you downloaded the release archive.*

**Remove all Rapid, Blitz and 960chess (9LX) games from the offline games**  
`tools/pgn-extract -t "tools/filters/notclassical.txt" -n "games/classical.pgn" "games/offline.pgn"`

games/classical.pgn is the file that contains all classical chess games after extraction  
games/offline.pgn is the input file that you want to extract the games from  

*this has already been done for you if you downloaded the release archive.*


Now we got a database with all classical games (classical.pgn).  
This DB can now be used to extract games from players so they can be processed efficiently with uci-analyser.  
For this we create a directory/folder for the player and create a file for each game that's processable by uci-analyser.  
The games have to be separated into black and white games for the player so uci-analyser only has to do half the work.  
I made a script that does the player extraction, directory creation and all the splitting of a .pgn file.  
This is how you would extract Nodirbek Abdusattorov:

`tools/split.sh "Abdusattorov,Nodirbek" "games/classical.pgn"`

It's important that the name is written exactly as it is in the pgn database.  
F.e "Carlsen,M" or "Ding Liren" or "Praggnanandhaa,R" or "Niemann,Hans Moke".

To find out how the names are written we can scan the .pgn with ripgrep:  
`rg -i "giri" "games/2019-2022.pgn" | sort | uniq -c`  
This shows that there are multiple Giris who's first name start with A ;-)

Hans Niemann is a special case (lol). He is listed as "Niemann,Hans" in earlier games and as "Niemann,Hans Moke" later on.  
This can be fixed with `sed` by first renaming everything to "Niemann,Hans" and then to "Niemann,Hans Moke"

`sed -i 's/"Niemann,Hans Moke"/"Niemann,Hans"/g' games/classical.pgn`  
`sed -i 's/"Niemann,Hans"/"Niemann,Hans Moke"/g' games/classical.pgn`

Now all Niemann games can be split using "Niemann,Hans Moke" as the player name.

You can check the games/players/ dir to see which players have already been extracted and split:  
`ls games/players`

Extract all the players you want and then we're ready to throw them at an engine of your choice.


### 3. Engine Analysis

This might take a lot of time. It takes about 3 hours for me to process ~4000 games with 24 threads using stockfish 14.

The default number of CPUs used is 8, you can override this by setting the ANALYSISBEADS_THREADS environment variable:  
`export ANALYSISBEADS_THREADS=24`  
this will use 24 CPUs instead of just the default 8

The script tools/analysisbeads.sh will process all dirs that are in the specified inputdir (default is games/players)  
It will write all the XML files containing the engine analysis into the specified output dir (default is results)  
The default engine is stockfish_14_x64_avx2 in the engines dir  
The default searchdepth is 20  
The default bookdepth is 0 (given in half moves)  
The default number of variations (multi engine lines) is 1 (only best move)  

`tools/analyse.sh [inputdir] [outputdir] [enginepath] [searchdepth] [bookdepth] [num_variations]`

so for example if you want to analyse all players in games/players with stockfish 8 using a depth of 12 and save the results in results_sf8

`tools/analyse.sh games/players results_sf8 engines/stockfish_8 12 0 1`

The script has a very crude progress meter, it tells you when a new player is started and at what time.  
Using this info you can estimate how long it will take to finish.



### 4. CPL (Centi-Pawn-Loss) Analysis

I wrote a script to extract average centipawn loss numbers (acpl.py)  
and one to extract centipawn losses for each move (cpl.py)  
They create a tab seperated values file (.csv) so data can easily be processed further.  

How to get acpl-data of a player sorted by date (includes event, result, enginecorrelation, etc)  
Manually for individual players:  
`rm -f temp/*`  the temp dir has to be cleared for parallel processing  
`find "results/Abdusattorov,Nodirbek/" -name "*.xml" -print0 | xargs -0 -n1 -P8 "tools/acpl.py"`    
`find "temp/" -name "*.xml" -print0 | sort -z | xargs -0 cat > "abdusattorov_acpl.csv"`  

There is a script to do this for all players and is much easier to use:  
`tools/acplbatch.sh results`  
This creates a .csv for each dir (aka player) in results.  


Same procedure for cpl-data for all moves in all games (games are in columns and moves are in rows).  
So moves are top to bottom and games are left to right.    
`rm -f temp/*`  
`find "results/Abdusattorov,Nodirbek/" -name "*.xml" -print0 | xargs -0 -n1 -P8 tools/cpl.py`  
`paste temp/* > abdusattorov_cpl.csv`

Again there is a script to do this for all players and is much easier to use:  
`tools/cplbatch.sh results`  

After watching the video of the brasilian guy (Rafael Leite) I also added a script to calculate the ACPL progression.  
`tools/acpl_trend.py niemann_acpl.csv`  

Output: 25.17   18.18

This shows that Niemann improved his ACPL over his last ~200 Elo gained (from 2019 to 2022) contrary to the claim in the video.



**REMARKS:**  
There is an issue with if the engine finds a mate in x moves. How much centipawn value does that have?  
PGNspy and the brasilian guy just used a flat value of 1000 or -1000, I dislike that approach but it might be a good one.  
What my script does is give 0 CPL if both the engine and the player move is a mate. The number of moves till mate is ignored (mate is mate after all).  
If the engine finds mate and the player doesn't or the player finds mate and the engine doesn't, the datapoint is thrown out (an empty cell).  
There is an extra column in the ACPL.csv indicating if the game had such an instance.  
The CPL.csv has an empty cell for this, which you need to be aware of if you are processing the csv,  
because some programs/libraries assume 0 on empty cell which distorts the results.

