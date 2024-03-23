---
title: "3D printed screws"
date: 2024-01-24T18:00:00+02:00
authors: ["Mikołaj Wilczek"]
---
My main reason for buying a 3D printer was to print mechanical parts - both for learning and creating home projects. 
Unfortunately, it took a while to start printing my own threads and screws. The main reason was the printer accuracy, which was insufficient for small threads for the early printers.

Today's printers have much better accuracy, so we are able to print screws and threads. On my MK4, I was able to print reasonable threads down to M4 diameter (I believe M3 is achievable too).
Most of the time, I'm using the thread command in Autodesk Fusion. 
For best results:
* Print them only towards the Z axis - the Z axis has the best accuracy (usually, you don't need to lower your layer height to make them work)
* If you need threads at different angles - make a hole for them and join them in a different way with the rest of the body (transparent, two-agent epoxy glue is the best for me).
* You can use metric screws in Fusion 360 - just remember to scale one of the parts a little. For me, it works best to scale down the screw to be smaller by 0.3 - 0.6mm than its original size (depending on the tolerances needed)
* **Do NOT scale Z axis** (along the screw) - after a few turns, you will not be able to turn it more.

I encourage you to use them as much as possible. They enable you to create strong, detachable solutions to join your 3D prints. Moreover, the newest public projects utilise them often.

Happy printing :)

## EDIT 23.03.2024
I've successfully tested printing M9 threads along the X and Y axes. Scaling it's screws to 95% gives a nice coupling on the MK4 printer.
