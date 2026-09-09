# Proton-reanimation
FE (FilteringEnabled)Primary: CurrentAngle V4 reanimation Fallback: FE-Reanimator V3 Second fallback: FE Headless / Nullware reanimation Rig focus: Primarily R6-style reanimation; the CurrentAngle settings explicitly have "R15 Reanimate" = false. Joint system: Motor6D-based character joint rebuilding/protection.
Features
MAX GODMODE — Raises the Humanoid's Health and MaxHealth to extremely high values and continuously restores health if it decreases.
Motor6D Protection — Creates and maintains important character joints, including the RootJoint, Neck, shoulders, and hips, helping prevent the character from becoming disassembled.
Anti-Fall — Saves the player's last safe position and attempts to teleport the character back if it falls into the void.
Anti-Fling — Monitors excessive character velocity and attempts to stop extreme movement.
ForceField Protection — Creates an invisible ForceField as another layer of character protection.
CurrentAngle V4 Reanimation — Attempts to load CurrentAngle V4 as the primary reanimation system.
FE-Reanimator V3 Fallback — If CurrentAngle fails, the script attempts to load FE-Reanimator-v3.
FE Headless Fallback — A third reanimation option is available if the previous systems fail.
Automatic Recovery — Detects character death/health loss and attempts to activate the reanimation system.
Draggable GUI — Includes a simple GUI with a button for starting the reanimation system.
Continuous Protection — Uses Heartbeat and property-change events to continuously monitor the character and reapply protection.
