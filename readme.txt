NextBASIC Shmup!
================


Kind of an attempt to do a classical style vertical shoot 'em up game, using new
features of NextBASIC. Born as a demo for me to learn the language, it grew up a
lot, and now we will see if it ends up being an actual game :)


How to load
-----------

1.- Copy the contents of this repo to your Next SD card.
2.- In your Next, go to its folder and type:
        .txt2bas shmup.bas.txt
3.- Load the newly created 'shmup.bas' and RUN it.


About emulators
---------------

Current version of CSpect (2.15.1) is not able to run the game. The previous
version (2.14.8) does run it but very slowly, about 50-75% of the speed on
actual hardware.


Motivation for this project
===========================

I wanted to do a game that achieves 2 things:

(1) Uses NextBASIC capabilities exclusively, so no calls to assembly, not even
an audio driver is used, just to test the capabilities of the language on its
own.

(2) Has "Dijkstra-compliant" code, with no GO TOs, and hopefully no line
numbers. This is difficult since makes me give up on GO SUBs and the
possibility to use 'computed GOSUBS' as "dictionaries", avoiding a good deal of
IF-THENs and improving performance. But I wanted to make readable code with a
proper representation of "modular programming", so if you read every PROC on
its own you should be able to understand everything.

I know I betrayed (2) on the setInput() procedure, but will be the only
exception ;)

From here on, I'll explain every trick the program implements.


Player movement
---------------

Movement of the player ship is not controlled "manually", there is no use of
variables for its position. Instead it uses a "finite-state machine" (FSM)
where the ship can be in 1 of 9 different states, and transition between states
occur by player input. Each state mean a different "automatic movement" type
for the ship, through SPRITE CONTINUE statement, which automagically takes care
of player position and limits of movement.

For info on this and other sprite manipulation commands, see
NextBASIC_New_Commands_and_Features.pdf in NextZXOS documentation, pages 7-11.
Direct link:
https://gitlab.com/thesmog358/tbblue/-/blob/master/docs/nextzxos
/NextBASIC_New_Commands_and_Features.pdf


FSM states
----------

State    Use
0        Standing still (neutral/no input)
1        Moving Right
2        Moving Left
3        Moving Up
4        Moving Down
5        Diagonal Up/Right
6        Diagonal Down/Right
7        Diagonal Up/Left
8        Diagonal Down/Left
9        Limbo (player dead, used to disable input)


Sound buffering
---------------

Sound was though out as a "circular queue", where %a() is the queue, %a is the
"head" or "front", which points to the next tone to play, and %u is the "tail"
or "rear" of the data, which is the last tone included in the queue.

Being in a circular queue, %a and %u only advance in one direction, and once
they reach the end, they go back to the beginning.


      %a                %u
       |                 |
       v                 ]
 *-----------------------------------------------------------------* <- %a()
 ^                                                                 |
 |_________________________________________________________________|


Any action that requires a sound fills the queue, advancing the "tail", and in
each SPRITE MOVE the "head" reads and plays the buffered sound, advancing to
the next position. If the "head" and the "tail" are in the same position,
nothing is played nor advanced (empty queue).

Note that there's no handling of "queue size", so no error checking if it's
completely "filled", it just start to overwrite old tones. In other words,
doing a full "turn" with the "tail" without making the "head" advance to catch
up will make the previously buffered sounds disappear.

%b(), %b and %v comprise a second queue, for a second audio channel. There are
other queues: coarse tone, %c(), and noise, %n(). These use the same head and
tail as the second channel queue, %b and %v, because they play through that
channel.


Sound playing
-------------

Only one AY is used, with "2 channels", since A and C play the same sound, and
their coarse setting is always 0. On the other side, B has coarse and fine, and
is able to produce noise when necessary.

Then, 4 elements are needed:
1.- Channel A & C fine tone - %a() - %a - %u
2.- Channel B fine tone     - %b() - %b - %v
3.- Channel B coarse tone   - %c() - %b - %v
4.- Noise (on or off)       - %n() - %b - %v

As said in the previous section, 4 circular queues are used, with the last 3
using the same "head" and "tail", %b and %v respectively.

To avoid changing the PSG mixer settings every time, previous mixer settings
are stored in %m, as a binary value with 0 = enabled and 1 = disabled:

                             0 0 0 N 0 A B A

...with N = noise
        B = B channel
        A = A & C channels

Then, %m = (16*N) + (5*A) + (2*B)

If you are wondering what is all of this, why it works this way, please read my
guide about generating sounds through OUT commands to the PSG:
https://www.specnext.com/forum/viewtopic.php?f=14&t=1796

At every SPRITE MOVE, queues are read, and from that the channels needed are
identified. If the enabled channels are different from what was stored in the
previous SPRITE MOVE, mixer settings are changed in the PSG and saved to %m.
Only after that, queues will advance and actual sound will be played.


Enemy wave spawning
-------------------

An "enemy wave" in this game's context is "enemies that spawn at the exact same
time". Having a short time between waves provides the illusion of a "coordinated
attack", while a longer time gives the pause one would expect from a "different
group of attackers".

DATA about enemies are READ by the wave() procedure in the following format:

- Number of enemies in the wave
- Time until next wave (in "game frames" - amount of SPRITE MOVE calls)
- Times the number of enemies, the following structure:
   - Enemy type
   - X position of spawn
   - Y position of spawn
- "Null terminator" (0).

"Enemy type" in this context is "how the enemy moves across the screen".
Position of spawn should always be outside of visible screen (BORDER coords).

RESTORE, which means spawning reset to the beginning, will occur when:
- "Number of enemies" and "Next spawn" are both 0
- "Null terminator" is not found when expected
Any of them could mean "end of stage" in the future. We'll see.

E.g. the first DATA in the program (near the end, after setInput() procedure)
contains the following:
                2, 40,     1, 130, 0,     1, 170, 0,    0

This indicates that this wave is composed of 2 enemies, and the next wave after
this will appear after 40 'SPRITE MOVEs'.
Then, in this wave the first enemy is type 1, and will appear at x=130, y=0.
The second enemy is also type 1 and will appear at x=170, y=0.
Then, the needed "0" to finish the reading of this wave.
Type 1 enemies descend vertically. This is why the first enemies you see on
screen are a couple at the top center, descending. You can check the
different 'types' at the "spawn()" procedure.


Boss
----

Currently, only 1 is allowed to exist at any time. It's composed of 4 unified
sprites with the following structure:
                                    2 3
                                    0 1

As sprites 1-3 have a pattern relative to the anchor sprite 0, a single SPRITE
CONTINUE to this anchor can be used for animation, since the relative sprites
inherit the pattern from the anchor, or rather, the sum of their original
pattern plus the anchor's pattern. This explains the order in which the sprites
are in the file 'shmup.spr', since adding numbers close to 0 provides control
on what the final patterns for every relative sprite will be.
Again, see NextBASIC_New_Commands_and_Features.pdf, pages 7 to 10


Variables
---------

Some are local for the procedure they appear in; they are shown here because of
their importance.

Name    Type        Use
%s      int         current State in the FSM
%i      int         current directional Input, FSM format
%c      int         player fire Cooldown (time before able to fire again)
%k()    int array   player controls (Keys or Kempston)
%l      int         Layer 2 position, for scroll effect
%f      int         movement Frames to wait before scroll
%m      int         current PSG Mixer settings
%w      int         needed mixer settings (to compare with %m)
%a()    int array   circular queue for A&C channels' sounds
%a      int         "head" of A queue
%u      int         "tail" of A queue
%b()    int array   circular queue for B channel's fine sounds
%c()    int array   circular queue for B channel's Coarse sounds
%n()    int array   circular queue for Noise on/off (B channel)
%b      int         "head" of B queues
%v      int         "tail" of B queues
%p()    int array   Player state (see below)
%d      int         player truly Dead (go to game over)
%e      int         time until next Enemy wave
%h      int         Hyperaggresiveness? Time until next shot from enemy
%o()    int array   bOss state (see below)


Player state
------------

%p() is an array with info about the player:

Index  Use
0      Dead (0), alive (1) or invulnerable (241)*
1      Lives
2      Score
3      Timer to respawn (used when dead)
       Also used to disable invulnerability after respawn
4      Cycles of enemy waves completed
       (times where DATA of enemy waves have been fully READ)

* Index 0, player vital status, affects player SPRITE flags, so colors are
inverted when player is invulnerable (just respawning).

      1 = 00000001 <- visible SPRITE, normal palette
    241 = 11110001 <- visible SPRITE, palette offset changed
   +  8 = 00001000 <- + X-mirrored, when going left


Boss state
----------

Likewise, %o() is an array with info about the boss:

Index  Use
0      Absent (0), type of movement (1-3)
1      Health
2      Time until next shot *

* The boss only shoots when its type of movement is 3


Sprite IDs
----------

Id      Use
48      Player - couldn't be any other id :)
49-57   Player shots
4-30    Enemies
31-47   Enemies' shots
60-77   Enemies' explosions/deaths
127     Player's death
0-3     Boss
126     Boss' death


Sprite patterns
---------------

Pattern Use
4       Player ship, straight
5       Player ship, leaning right
16      Player shot
6-11    Explosion animation
12-15   Enemies and their animations
0-3     Half of boss character and animation
17      Enemy shot
18      Boss shot (to do)


To Do
-----

Have to make it playable!
- Enemies shooting in different directions
- Make boss to shoot
- Other types of waves
- Powerups?
- Improved ending? Change its condition?


Acknowledgments
===============

The entire Next Team, for the wonderful machine
Garry Lancaster, for the language and the Invaders game which inspired this
Remy Sharp, for the info about audio and Layer 2
Simon N Goodwin and Matthew Neale, for performance tips
Rodrigo Moreira, for the beautiful sprites :)
You, for your interest in the game and getting this far in the document

Jaime Moreira
2020-08-23
Last updated: 2022-03-20
