NFL Play-by-Play Data (2002-2012)
=========

This is a refinement of the NFL play-by-play data released by Brian Burke of Advanced NFL Stats: http://www.advancednflstats.com/2010/04/play-by-play-data.html

nflstats.csv is a tab-delimited CSV with all plays from 2002-2012 (including the postseason for 2002-2011).  The file is about 77MB in size and contains 471,583 plays.

The column names are in the first row of the document and are mostly self-explanatory:


Data
-----
- `PLAY_ID`: A unique integer play ID
- `GAME_ID`: A unique game ID of the format `YYYYMMDD_AWAYTEAM@HOMETEAM`
- `QTR`: What quarter of the game the play occurs in
- `MIN`: How many minutes are left in the game (from 60)
- `SEC`: How many seconds are left in the game in addition to `MIN`
- `OFF`: Which team is on offense
- `DEF`: Which team is on defense
- `DOWN`: What down it is
- `YARDS_TO_FIRST`: Yards needed for offense to get a first down/touchdown
- `YARDS_TO_GOAL`: Yards needed for the offense to get a touchdown
- `DESCRIPTION`: A description of the play (more information below)
- `OFF_SCORE`: Current points for the offense
- `DEF_SCORE`: Current points for the defense
- `SEASON`: Calendar year for the season (postseason games count for the year before if in January/February)
- `YEAR`: Year of the game date
- `MONTH`: Month of the game date
- `DAY`: Day of the game date
- `HOME_TEAM`: Which team was the home team
- `AWAY_TEAM`: Which team was the away team
- `PLAY_TYPE`: a series of identifiers separated by pipe symbols (|) explaining what kind of play it is.  A play can have any number of identifiers, or zero.

Description/Play Type
-----
The main problem with this data is that almost all of the information about the play is stored in the `DESCRIPTION` field, for example:

	M.Cassel pass short middle to C.Chambers pushed ob at PIT 4 for 61 yards (W.Gay).
	(5:13) R.Dayne left end to HST 17 for 4 yards (K.Bulluck D.Thornton).
	J.Tuthill kicks 59 yards from WAS 30 to GB 11. J.Walker to GB 37 for 26 yards (K.Watson). PENALTY on GB-T.Franz  Offensive Holding  10 yards  enforced at GB 37.

While the descriptions are detailed, they're also extremely messy and inconsistent.  The format varies wildly.  There's stray punctuation.  Some of the rows have garbled data instead.  And football plays can be extremely complex.  You can have a pitch, an interception, a fumble, a touchdown, and a replay review, all on the same play, and the way such a play is described is totally unpredictable.

In an effort to extract at least some consistent data from these plays, I explored different text patterns and play type hierarchies and come up with the following list of `PLAY_TYPE` identifiers:

	RUN
	RUN_LEFT
	RUN_RIGHT
	RUN_MIDDLE
	QB_SNEAK
	DRAW
	FUMBLE
	INTERCEPTION
	PASS_INCOMPLETE
	PASS_COMPLETE
	TOUCHDOWN
	FIELD_GOAL_GOOD
	FIELD_GOAL_MISSED
	FIELD_GOAL_BLOCKED
	CONVERSION_GOOD
	CONVERSION_FAILED
	PUNT
	PUNT_BLOCKED
	PUNT_FAKE
	FIELD_GOAL_FAKE
	KICKOFF
	SAFETY
	SACK
	PENALTY
	NO_PLAY
	TURNOVER
	CHALLENGE
	CHALLENGE_SUCCESSFUL
	CHALLENGE_FAIL

A play can have any number of `PLAY_TYPE` identifiers in the field, separated by pipe symbols (|), for example:
	(10:07) C.Benson right end to DEN 39 for 6 yards (M.Haggan).	RUN|RUN_RIGHT
	(1:21) T.Brady pass intended for C.Cleeland INTERCEPTED by B.Robinson at CHI 35. B.Robinson to CHI 37 for 2 yards (M.Compton). FUMBLES (M.Compton) recovered by CHI-R.Colvin at CHI 37. Play Challenged by Review Assistant and REVERSED. T.Brady pass incomplete to C.Cleeland (B.Robinson).	FUMBLE|INTERCEPTION|PASS_INCOMPLETE|CHALLENGE|CHALLENGE_SUCCESSFUL 	

Caveats
-----

While most of the `PLAY_TYPE` values are pretty accurate, there are plenty of strange plays that break the patterns.  This is meant as a best guess and is definitely not 100% correct.  One particular large category that's excluded is run plays that don't specify anything about the direction or type of run.  Some run plays are of this format and are very hard to match with rules, like:

	(10:51) P. Holmes to OAK 13 for 1 yard (T. Bryant).

To Do
-----

- Add kneeldowns, spikes, and extra points to `PLAY_TYPE`
- Figure out a good regular expression to recognize the generic run descriptions
- Add a `DRIVE_ID` field to group plays easily by drive

Conditions for PLAY_TYPE
-----

Below are the MySQL conditionals I used to assign PLAY_TYPE values:

	ASSIGN...				WHERE...
	RUN						DESCRIPTION LIKE "%up the middle%" OR DESCRIPTION REGEXP "(left|right) (tackle|guard|end)" OR DESCRIPTION REGEXP " rushe(d|s) for "
	RUN_LEFT				DESCRIPTION REGEXP "left (tackle|guard|end)"
	RUN_RIGHT				DESCRIPTION REGEXP "right (tackle|guard|end)"
	RUN_MIDDLE				DESCRIPTION LIKE "%up the middle%"
	QB_SNEAK				DESCRIPTION LIKE "%sneak%"
	DRAW					DESCRIPTION REGEXP " draw[} .-]" OR DESCRIPTION LIKE "%draw"
	FUMBLE					DESCRIPTION LIKE "%fumble%"
	INTERCEPTION			DESCRIPTION LIKE "%intercept%"
	PASS_INCOMPLETE			DESCRIPTION REGEXP " pass(ed|es|[.])? " AND DESCRIPTION LIKE "%incomplete%"
	PASS_COMPLETE			DESCRIPTION REGEXP " pass(ed|es|[.])? " AND DESCRIPTION NOT LIKE "%intercept%" AND DESCRIPTION NOT LIKE "%incomplete%"
	TOUCHDOWN				DESCRIPTION LIKE "%touchdown%"
	FIELD_GOAL_GOOD			DESCRIPTION LIKE "%field goal%" AND DESCRIPTION NOT LIKE "% no play%" AND DESCRIPTION LIKE "% good%" AND DESCRIPTION NOT LIKE "%no good%"
	FIELD_GOAL_MISSED		DESCRIPTION LIKE "%field goal%" AND DESCRIPTION NOT LIKE "% no play%" AND DESCRIPTION NOT LIKE "% blocked%" AND DESCRIPTION REGEXP "(short|wide left|wide right|no good)"
	FIELD_GOAL_BLOCKED		DESCRIPTION LIKE "%field goal%" AND DESCRIPTION NOT LIKE "% no play%" AND DESCRIPTION LIKE "% blocked%"
	FIELD_GOAL_FAKE			(DESCRIPTION LIKE "%field goal%" AND DESCRIPTION LIKE "% fake%") OR (DESCRIPTION LIKE "%field goal formation%" AND DESCRIPTION NOT LIKE "% good%" AND DESCRIPTION NOT LIKE "% blocked%" AND DESCRIPTION NOT LIKE "% no play%")	
	CONVERSION_GOOD			DESCRIPTION LIKE "%conversion%" AND DESCRIPTION LIKE "%succeeds%"
	CONVERSION_FAILED		DESCRIPTION LIKE "%conversion%" AND DESCRIPTION LIKE "%fails%"
	PUNT					DESCRIPTION REGEXP " punt(s|ed)"  AND (DESCRIPTION NOT LIKE "%blocked%" OR DESCRIPTION LIKE "%partially blocked%")
	PUNT_BLOCKED			DESCRIPTION REGEXP " punt(s|ed)" AND DESCRIPTION LIKE "%blocked%" AND DESCRIPTION NOT LIKE "%partially%"
	PUNT_FAKE				(DESCRIPTION REGEXP " punt(s|ed)"" AND DESCRIPTION LIKE "% fake%") OR (DESCRIPTION LIKE "%punt formation%" AND DESCRIPTION NOT REGEXP " punt(s|ed) ")
	KICKOFF					DESCRIPTION REGEXP " kick(s|ed) "
	SAFETY					DESCRIPTION LIKE BINARY "%SAFETY%"
	SACK					DESCRIPTION LIKE "%sacked%"
	PENALTY					DESCRIPTION LIKE "%penalty%" AND DESCRIPTION NOT LIKE "%no penalty%"
	NO_PLAY					DESCRIPTION REGEXP " no play[ .}-]"
	TURNOVER				DESCRIPTION LIKE "%fumble%" OR DESCRIPTION LIKE "%intercept%"
	CHALLENGE				DESCRIPTION LIKE "%challenge%" AND DESCRIPTION NOT REGEXP "(unchallengable|not challengable|not reviewable|unreviewable|inoperable|cannot be challenged|review equipment|malfunction|none remaining|prevent challenge)"
	CHALLENGE_SUCCESSFUL	DESCRIPTION LIKE "%challenge%" AND DESCRIPTION NOT REGEXP "(confirmed|upheld|not overturned)" AND DESCRIPTION NOT REGEXP "(unchallengable|not challengable|not reviewable|unreviewable|inoperable|cannot be challenged|review equipment|malfunction|none remaining|prevent challenge)"
	CHALLENGE_FAIL			DESCRIPTION LIKE "%challenge%" AND DESCRIPTION REGEXP "(confirmed|upheld|not overturned)"

Contact
-----
Noah Veltman
noah@noahveltman.com
http://noahveltman.com/