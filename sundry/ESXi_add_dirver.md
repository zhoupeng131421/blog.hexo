1. download ESXi-Customizer
    - git clone https://github.com/VFrontDe/ESXi-Customizer-PS.git
2. open powershell with Administrator
3. install powershell module
    - Install-Module -Name VMware.PowerCLI​​​
    - A / Y
    - wait ...
4. online add r8125
    - .\ESXi-Customizer-PS.ps1 -v67 -load net-r8125
    - wait ...