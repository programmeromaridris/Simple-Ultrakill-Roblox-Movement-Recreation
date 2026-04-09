# Simple-Ultrakill-Roblox-Movement-Recreation
This script seeks to recreate popular Indie title 'Ultrakill''s movement system, a high-mobility character controller using ContextActionService for input binding. It manages a modular and scalable set of movement abilities focused on momentum 


Modular Actions: Binds Sprint, Dash, Jump, Slide, Groundslam, Parry and M1 attacks
Sliding: Reduces friction and lowers HipHeight for a steerable, drifting slide using BodyVelocity.
Better Jumping: Handles multi-jumps and raycast-based vaulting using AssemblyLinearVelocity.
Dashing: Adds bodyvelocity to the player's character to propel them forward in any direction (except diagonal)
GroundSlam: Downwards bodyvelocity, along with spawning a hitbox at the players feet.

Parry: Fires a server event telling the enemyhandler that the player is parrying, and negate all enemy attacks.
M1: Fires a server event, basically. Along with some extra functions to detect if the weapon is auto, which allows for holding the M1 button.
State Management: Centralized PlayerState module tracks cooldowns, animation states, and action availability.
Automatic Cleanup: Includes landing detection to reset jump counters and internal routines to prevent memory leaks.
