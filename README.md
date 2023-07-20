# DF-RFeFET_2023_CSM
## Introduction
  + 이 시뮬레이션은 Synopsys Senstaurus version O-2018.06. 사용했습니다.
  + 이 시뮬레이션에서 Ferroelectric hysteretic 특성을 재현하고 FeFETs 시뮬레이션도 수했습니다.
  + FeFET의 기본 시뮬레이션과 DF-RFeFET에 대한 간략한 설명은 여기에 별도로 제시됩니다.
## FeFET 기본 시뮬레이션 설명
  + structure: sentaurus--sde tool
  + sdevice: Ferroelectric material parameter
 	 + Use a material called  *InsulatorX*  to simulate the Ferroelectric hysteretic characteristic.
  	 + Use *epsilon = 25* to represent the Ferroelectric material multi-domain switching.
 	 + The **Preisach model** is used in this simulation, and following this model we need to define the *Polarization* parameter   *(more detail in sentaurus/manual/sdevcie_ug/PartⅡ Physics in Sentaurus Device/Chapter 29 Ferroelectric Materials)*.
 	
  ```
Material="InsulatorX" {
  #includeext "ParFileDir/InsulatorX.par"
 Epsilon
{ *  Ratio of the permittivities of material and vacuum
  * epsilon() = epsilon
	epsilon	= 25 	# [1]
}
Polarization
{ * Remanent polarization P_r, saturation polarization P_s, 
  * and coercive field F_c for x,y,z direction (crystal axes)
 P_r = (0, 2e-05, 0) #[C/cm^2]
 P_s = (0, 4e-05, 0) #[C/cm^2]
 F_c = (0, 0.67e+06, 0) #[V/cm]
* Relaxation time for the auxiliary field tau_E, relaxation
* time for the polarization tau_P, nonlinear coupling kn.
tau_E = (0.0000e+00,0.0000e+00, 0.0000e+00) #[s]
tau_P = (0.0000e+00,250e-9, 0.0000e+00) #[s]
kn = (0.0000e+00, 0.0000e+00, 0.0000e+00) #[cm*s/V]
}}
  ```
  * sdevice: Id-Vg output method
 	  ![image](https://github.com/sami1016/DF-RFeFET_2023_CSM/blob/main/voltage.png) [1]
	 * The Id-Vg curves are simulated by using a read Vg sweep after the program and erase operation, like the upper picture.
 	 * Firstly, we set up a sdevice tool called *sdevice1* to perform the program/erase operation.
  	 * program/erase operation code:
   		 * Use the *Transient simulation* method to do program/erase operation.
     		 * *@dtp@* is program voltage duration time, *@dte@* is erase voltage duration time.
       		 * *@Vgp@* is program voltage duration time, *@Vge@* is erase voltage duration time.
  ```
Electrode {
	{Name="Gate" 	Voltage= 0   workfunction= 4.7
	Voltage=(
	!(
         set t1e 0
	 for { set i 1 } { $i <= @cycles@ } { incr i } { 
	   set t2e [expr $t1e + 1e-9]
	   set t3e [expr $t1e + (@dtp@*1e-6)]	   
	   set t4e [expr $t2e + (@dtp@*1e-6)]
	   	   
	   set t1p [expr $t3e + 1e-3 ]
	   set t2p [expr $t1p + 1e-9]
	   set t3p [expr $t1p + (@dte@*1e-6)]
	   set t4p [expr $t2p + (@dte@*1e-6)]
    puts "	0 at [format %0.6e $t1e], @Vge@ at [format %0.6e $t2e], @Vge@ at [format %0.6e $t3e], 0 at [format %0.6e $t4e],"
    puts "      0 at [format %0.6e $t1p], @Vgp@ at [format %0.6e $t2p], @Vgp@ at [format %0.6e $t3p], 0 at [format %0.6e $t4p],"
	   set t1e [expr $t3p + 1e-3]
	 }
	  )!
	 )
	}   
	{Name="Substrate" Voltage= 0}
	{Name="Drain"	Voltage= 0 Distresist= 1e-6}
        {Name="Source"	Voltage= 0}}
  ```
		*And in sdevice1 Solve part, we separately store the states of the program and erase opreation.
  ```	
Solve {
# initial solution
    	coupled (LineSearchDamping= 0.01) { Poisson }

	Plot (FilePrefix="n@node@_Zero")
	Transient(
	    InitialTime= 0 FinalTime= !(puts [expr $t1e])! 
	    InitialStep= 1e-4 Minstep= 1e-25 MaxStep= 5e-3
	){ Coupled { Poisson Electron Hole } 

      !(
      	puts "Plot(FilePrefix=\"n@node@_erased\" Time=([format %0.6e [expr $t3e]]))"
	puts "Plot(FilePrefix=\"n@node@_erased_no_bias\" Time=([format %0.6e [expr $t1p]]))"
	puts "Save(FilePrefix=\"n@node@_erased\" Time=([format %0.6e [expr $t3e]]))" 
      
	puts "Plot(FilePrefix=\"n@node@_programmed\" Time=([format %0.6e [expr $t3p]]))"
	puts "Plot(FilePrefix=\"n@node@_programmed_no_bias\" Time=([format %0.6e [expr $t1e]]))"
	puts "Save(FilePrefix=\"n@node@_programmed\" Time=([format %0.6e [expr $t3p]]))" 
        )!
	}
		}
  ```
		* Secondly, we set up a sdevice tool called *Iv* to perform the read operation.
  		* read operation code:
  ```
   Electrode {
	{Name="Gate" Voltage= 0  workfunction= 4.7
	Voltage=(
	 0 at 0, 
	-3 at 1e-6,
	 5 at 5e-3)
	 }      
	{Name="Substrate" Voltage= 0}
	{Name="Source"	Voltage= 0 }
	{Name="Drain"	Voltage= 0 Voltage= (0 at 0, @Vd@ at 1e-6)  Distresist= 1e-6  }
}
  ```
		*And in Iv Solve part, we get the states of the program/erase operation and read the Id-Vg.
 ```
Solve {
        *- Build-up of initial solution:
   coupled (LineSearchDamping= 0.01) { Poisson}
		    
	Transient (
	   InitialTime= 0 FinalTime= 1e-6
    	   InitialStep= 1e-11 MaxStep= 5e-7 Minstep= 1.e-20
	){ Coupled {Poisson Electron Hole}
           Load(FilePrefix="n@node|sdevice1@_@state@" Time=(0))
          }
  
	NewCurrentFile="n@node@_IdVg_"
	Transient (
	   InitialTime= 1e-6 FinalTime= 5e-3
	   InitialStep= 1e-9 MaxStep= 3.125e-02 Minstep= 1e-20
	){ Coupled {Poisson Electron Hole}
           CurrentPlot( Time=(Range=(0 5e-3) Intervals= 60) )
	}	}
  ```
## DF-RFeFET 설명
* In the DF-RFeFET paper, we confirm the performance of the DF-RFeFET through model-calibrated TCAD simulations and observe that it exhibits a larger memory window (MW). Additionally, by optimizing the device, we achieve a large MW of 5.5V. We believe that the DF-RFeFET is a promising candidate for future memory applications, providing valuable insights for further advancements in FeFET technology.
* More detail about DF-RFeFET in [doc](https://github.com/sami1016/DF-RFeFET_2023__CSM/tree/main/doc)
* More simulation detail in [project](https://github.com/sami1016/DF-RFeFET_2023__CSM/tree/main/project),we set many parameters more than in the lecture
	* Fig.1:[MFM_P-E](https://github.com/sami1016/DF-RFeFET_2023__CSM/blob/main/project/mfm.gzp),[Planar FeFET_Id-Vg](https://github.com/sami1016/DF-RFeFET_2023__CSM/blob/main/project/planar.gzp)
  	* Fig.4:[DF-planar FeFET](https://github.com/sami1016/DF-RFeFET_2023__CSM/blob/main/project/DF-planar.gzp), [SF-RFeFET](https://github.com/sami1016/DF-RFeFET_2023__CSM/blob/main/project/SF-RFeFET.gzp),[DF-RFeFET](https://github.com/sami1016/DF-RFeFET_2023__CSM/blob/main/project/DF-RFeFET.gzp)
 	* Fig.5:[DF-RFeFET](https://github.com/sami1016/DF-RFeFET_2023__CSM/blob/main/project/DF-RFeFET.gzp)
 	* Fig.6:[DM-RFeFET](https://github.com/sami1016/DF-RFeFET_2023__CSM/blob/main/project/DM-RFeFET.gzp),[DF-RFeFET_depth](https://github.com/sami1016/DF-RFeFET_2023__CSM/blob/main/project/DF-RFeFET_depth.gzp)
  	* Fig.7:[intermediate layer](https://github.com/sami1016/DF-RFeFET_2023__CSM/blob/main/project/Intermediate.gzp)
  	* Fig.8:[EOT=1.2](https://github.com/sami1016/DF-RFeFET__2023__CSM/blob/main/project/HK_sio2_1.2.gzp),[EOT=0.7](https://github.com/sami1016/DF-RFeFET_2023__CSM/blob/main/project/HK_sion_0.7.gzp),[EOT=0.54](https://github.com/sami1016/DF-RFeFET_2023__CSM/blob/main/project/HK_sio2hfo2_0.54.gzp)
## Reference
[1] Mulaosmanovic,Halid,et al."Ferroelectric field-effect transistors based on HfO<sub>2</sub>: a review".Nanotechnology,vol.32,2021,pp.502002,https://doi.org/10.1088/1361-6528/ac189f.
