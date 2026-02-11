# Simple-Ultrakill-Roblox-Movement-Recreation
This script seeks to recreate popular Indie title 'Ultrakill''s movement system,  a high-mobility character controller utilizing ContextActionService for cross-platform input binding. It 
manages a modular suite of movement abilities focused on momentum and physics-driven traversal.


Modular Actions: Binds Sprint, Dash, Jump, Slide, and M1 attacks to PC, Console, and Mobile.
Physics-Based Sliding: Reduces friction and lowers HipHeight for a steerable, drifting slide using BodyVelocity.
Advanced Jumping: Handles multi-jumps and raycast-based vaulting using AssemblyLinearVelocity.
State Management: Centralized PlayerState module tracks cooldowns, animation states, and action availability.
Automatic Cleanup: Includes landing detection to reset jump counters and internal routines to prevent memory leaks.
