Test simulation case 6 in ~/MiV-Simulator-Cases/ using ~/MiV-Simulator, ~/neuroh5~/core, and ~/bin/miv_iowarp_bench.pbs.
Since ~/core is udpated, update ~/bin/miv_iowarp_bench.pbs if necessary.
Fix any build and test errors on ~/MiV-Simulator and ~/neuroh5 and ~/core.
Your goal is to compare the performance of using ~/core or not.
Use g++-14 for building and testing.
Do not use GPUs.
Start with 1 node using debug queue.
If 1 node is successful, 
run maximum number of nodes that gpu_hack_large queue allows.
If gpu_hack_large is not available use preemtable queue instead.
Summary simulation performance result in an .md file.
Update ~/NeuroFAIR/wiki based on the test results.
Use envs under ~/miniconda3/ for dependency download sand linking.
