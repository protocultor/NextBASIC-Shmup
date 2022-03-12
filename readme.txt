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

Only one AY will be used, with "2 channels", since A and C will play the same
sound, and their coarse setting will always be 0. On the other side, B will have
coarse and fine, and will be able to produce noise when necessary.

Then, 4 elements are needed:
1.- Channel A & C fine tone - %a() - %a - %u
2.- Channel B fine tone     - %b() - %b - %v
3.- Channel B coarse tone   - %c() - %b - %v
4.- Noise (on or off)       - %n() - %b - %v

As said in the previous section, 4 circular queues will be used, with the last
3 using the same "head" and "tail", %b and %v respectively.

To avoid changing the PSG mixer settings every time, we will store previous
settings in %m, as a binary value with 0 = enabled and 1 = disabled:

                             0 0 0 N 0 A B A

...with N = noise
        B = B channel
        A = A & C channels

Then, %m = (16*N) + (5*A) + (2*B)

If you ask yourself what is all of this, why it works this way, please read my
guide about generating sounds through OUT commands to the PSG:
https://www.specnext.com/forum/viewtopic.php?f=14&t=1796

At every SPRITE MOVE, queues are read, and from that the channels needed are
identified. If the enabled channels are different from what was stored in the
previous SPRITE MOVE, mixer settings are changed in the PSG and saved to %m.
Only after that, queues will advance and actual sound will be played.


Enemy waves
-----------

They're in DATA statements, handled by the wave() procedure. First data is the
amount of enemies in this wave, second is the time to read the next wave, with
"time" being the amount of SPRITE MOVEs called (see playGame() procedure).
After that, every enemy is a trio, which indicate type, x-coord and y-coord.
"Type of enemy" in this context is "how the enemy moves in the screen".
To finish the wave, there must be a 0.

E.g. the first DATA in the program (line 7 approx.) contains the following:
                  2,40,    1,130,0,    1,170,0,   0
This indicates that this wave is composed of 2 enemies, and the next wave after
this will appear after 40 'time' has passed.
Then, in this wave the first enemy is type 1, and will appear at x=130, y=0.
The second enemy is also type 1 and will appear at x=170, y=0.
Then, the needed "0" to finish the reading of this wave.
Type 1 enemies descend vertically. This is why the first enemies you see on
screen are a couple at the top center, descending. You can check the
different 'types' at the "spawn()" procedure.


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


Player state
------------

%p() is an array with info about the player:

Index  Use
0      Alive (1) or dead (0)
1      Lives
2      Score
3      Timer to respawn (used when dead)
       Also used to remove invulnerability after respawn


Sprite IDs
----------

Id      Use
48      Player - couldn't be any other id :)
49-57   Player shots
1-30    Enemies
31-47   Enemies' shots
60-77   Enemies' explosions/deaths
127     Player's death


Sprite patterns
---------------

Pattern Use
0       Player ship, straight
1       Player ship, leaning right
2       Player shot
3-8     Explosion animation
9-13    Enemies and their animations


To Do
-----

Have to make it playable!
- Make enemies shoot
- Other types of enemies
- Powerups? Bosses?


Acknowledgments
---------------

The entire Next Team, for the wonderful machine
Garry Lancaster, for the language and the Invaders game which inspired this
Remy Sharp for the info about audio and Layer 2
You, for your interest in the game and getting this far in the doc :)

Jaime Moreira
2020-08-23
Last updated: 2022-03-12
