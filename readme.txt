NextBASIC Shmup!
================


Kind of an attempt to do a classical style vertical shoot 'em up game, using new
features of NextBASIC. Born as a demo for me to learn the language, it grew up a
lot, and now we will see if it ends up being an actual game :)


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
%o      int         scOre
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


Sprite IDs
----------

Id      Use
48      Player - couldn't be any other id :)
49-57   Player shots
1-30    Enemies (?)
31-47   Enemies' shots (??)
60-77   Enemies' explosions/deaths


Sprite patterns
---------------

Pattern Use
0       Player ship, straight
1       Player ship, leaning right
2       Player shot
3-4     Enemy
5-10    Explosion animation


To Do
-----

Have to make it playable! Spawn enemies, make them shoot, make the player able
to die. Have to provide a challenge of some sort.


Acknowledgments
---------------

The entire Next Team, for the wonderful machine
Garry Lancaster, for the language and the Invaders game which inspired this
You, for your interest in the game and getting this far in the doc :)

Jaime Moreira
2020-08-23
