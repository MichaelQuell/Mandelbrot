Parallel Mandelbrot
---

###Introduction
---

This project creates an image of a [Mandelbrot fractal](http://en.wikipedia.org/wiki/Mandelbrot_set). This repository contains three seperate codes: a serial version, an OpenMP version, and a hybrid OpenMP and MPI version.

This project uses the [Distance Estimator](http://mrob.com/pub/muency/distanceestimator.html) method to calculate the Mandelbrot set. Each pixel requires its own independent unit of work. Because these units of work are indepenent, it is a prime canidate for parallelizaion. 

The code will perform an iterative calculation at each point that will determine weather if the point [escapes the set](http://en.wikipedia.org/wiki/Mandelbrot_set#Formal_definition). If the point doesn't escape, then it is considered part of the Mandelbrot set and the pixel is painted black.

The Distance Estimator comes into play if the point escapes very slowly (takes many iterations before it escapes). For visualization purposes, we consider these points to be part of the set. By doing this, it will "reveal" more of the set than the standard method. 

####Serial
This code is located in `serial_mandelbrot.c`. At the beginning of the serial code, a 2 dimensional grid is allocated in memory. This grid will hold either a 1 or 0 in each element to represent wheather a point is in the set or not. 

The code then enters a nested loop region that will iterate over each point (element in the grid) and perform the iterative Distance Estimator calculation. When the calculation determines if the point escapes, it breaks out of the loop and writes a "0" in the grid for that point. Else, it will infer that the point does not escape and will write a "1" in the grid. 

After iterating over each point, the grid is then passed to a function (in `tiff.c`) that will write out a TIFF image to the current working directory.

####OpenMP
The OpenMP parallelized version of this code is availible in the file `openmp\_mandelbrot.c.` The code here is nearly identical to the serial version except for one very important difference. As you can see on line 45 it reads
```C
#pragma omp for
```
This tells the compiler to seperate the enclosed `for()` loop's iterations and run them in parallel on different threads. This is possible because the calculation for each point is independant on any of the surrounding points' calculations.

Again, once the calculations are complete, the same function is called to write the image out.

####Hybrid OpenMP and MPI
This code is available in the file `openmp_mpi_mandelbrot.c` At the beginning, the MPI instance is created by calling
```C
MPI_Init(&argc, &argv);
```
Again, a two dimensional grid is allocated. Next, the image is broken up into sets of rows based on the number of MPI ranks minus one. Rank 0 will then send the starting row number and ending row number as well as the size of total points to be calcuated upon to the appropriate process. 

On each compute rank (all processes except rank 0) a smaller grid is allocated based upon the total number of points to be calculated (or `chunkSize`). 

The points are then iterated over the same way as in the OpenMP version. While the calcuations are going on on ranks 1-n, rank 0 is waiting to recieve the completed chunks of the image. 

Once the calculation is complete, that rank will send the grid to rank 0 and will then be copied onto the large grid at the appropriate place, based upon the sending rank number. 

Once all processes have finished and sent their respective grids, rank 0 will finish assembling the overall image grid and pass it to the image writing funcion defined in `tiff.c`. The image will then be written out to the current directory. 

Finally the MPI instance will be ended by calling
```C
MPI_Finalize();
```

###Building libTIFF
---
libTIFF must be compiled and availible in your home directory (for Titan). You can skip this section if you are compiling and running on Rhea. If you need to build libTIFF yourself, use the Ruby script in the buildTiff directory.

```
./buildTiff.rb
```

###Compiling on Titan
---
Before you can compile on Titan, the makefile needs to be edited and < PROJID > needs to be changed to the appropriate project id. To compile run...

`make titanSerial` (for serial version)

`make titanOMP` (for OpenMP parallelized version)

`make titanHybrid` (for hybrid OpenMP and MPI parallelized version)

###Compiling on Rhea
---
Before you can compile on Rhea, the makefile needs to be edited and < PROJID > needs to be changed to the appropriate project id. To compile on Rhea run...

`make rheaSerial` (for serial version)

`make rheaOMP` (for OpenMP parallelized version)

`make rheaHybrid` (for hybrid OpenMP and MPI parallelized verion)

NOTE: You can use the rules for compiling on Rhea on other general clusters that have libTIFF already installed in the default location.

###Running
---
This can be run from either an interactive job or by submitting a batch script. Make sure to use the correct batch script on the appropriate machine (titan.pbs for titan and rhea.pbs for rhea). Before submitting either batch script, edit the script and change the 2 instances of < PROJID > to your appropriate project id. After doing that, save and then run

`qsub titan.pbs`

for Titan. To run on Rhea, simply run

`qsub rhea.pbs`

This can also be run from an interactive job. To do this, run the command

```
qsub -I -V -A< PROJID > -lnodes=< number of nodes needed >,walltime=2:00:00
```
Don't forget to replace `< PROJID >` and `< number of nodes needed >` with your project ID and number of nodes, respectively. When in your interactive job, run

```
aprun -n<number of processes> ./mandelbrot
```

or on Rhea, run

```
mpirun -n<number of processes> mandelbrot
```

again replacing < number of processes > with the number of processes you want to run. Running this will create a TIFF image in your $MEMBERWORK/< PROJID > directory.

The resolution can be changed by editing the `nx` and `ny` variables and changing `maxiter` will change how many iterations are performed before determining if the point escapes or not. 

The image will be called `output.tif` and will look like this:

![The Mandelbrot](https://raw2.github.com/JRWynneIII/Mandelbrot/master/example.png)

