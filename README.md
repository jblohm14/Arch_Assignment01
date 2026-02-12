# Joshua Blohm - Assignment 1

## 1. Description
For this assignment I chose to analyze a library used to compute Fourier transforms called [fftw3](https://github.com/FFTW/fftw3). I have used this library in my profession as an embedded systems engineer. I use this library because it uses bootstrap strategies to optimize itself depending on the hardware it’s running on. It is very popular because it is open-sourced, efficient, and has a small code footprint. It is a suitable repository for hotspot analysis because it has 587 files and over 3,000 commits.

## 2. Timeframe
The library fftw3 is somewhat old. Its most recent release was on Oct 29, 2017. To investigate which files have paid interest on technical debt I chose the timeframe beginning with the date of the last release through the present. I chose a timeframe of several years to capture all changes made since the last release. Hotspots identified in this timeframe may be the result of technical debt. I reformatted the git logs for the fftw3 repository so that _maat_ could ingest the metadata using the command below and store the output as a log file.  
```
git log --pretty=format:'[%h] %an %ad %s' --date=short --numstat --before=2025-12-31 --after=2017-10-29 > fftw3_evo.log
```
## 3. Analysis
To analyze the fftw3 repository for hotspots I used the `maat` and `cloc` tools provided in the provided lab tools.   

1. First, I analyzed the fftw3 repo using a measure of how often the file changes to gauge effort spent. I used the command below to parse the formatted SCM logs and capture the commit frequency of all changed files within my timeframe.
    ```
    $ maat -l fftw3_evo.log -c git -a revisions > fftw3_freqs.csv
    ```
    Below are the first 10 entries showing the most frequently changed files. I didn't expect many changes but it's interesting that the file `configure.ac` changed 17 times since the official release. 
    ```
    configure.ac,17 🡄
    simd-support/simd-maskedsve.h,12
    CMakeLists.txt,11
    tests/fftw-bench.c,8
    simd-support/simd-common.h,6
    kernel/cycle.h,6
    NEWS,6
    simd-support/Makefile.am,5
    kernel/ifftw.h,5
    Makefile.am,4
    ```

1. Next, I measured the library using a simple metric, lines of code (LOC), to gauge complexity using this command:
    ```
    $ cloc ./ --unix --by-file --csv -quiet --report-file=fftw3_lines.csv
    ```
    Below are the first 10 entries showing the most complex files. I noticed the most changed file from the previous metric is also the 6th most complex with 686 lines of code.
    ```
    C,./libbench2/verify-r2r.c,121,63,780
    C/C++ Header,./kernel/ifftw.h,187,210,754
    C,./mpi/mpi-bench.c,80,12,752
    C,./kernel/planner.c,172,128,735
    C,./mpi/api.c,115,82,710
    m4,./configure.ac,124,68,686 🡄
    C,./rdft/vrank3-transpose.c,106,124,547
    C,./libbench2/mp.c,105,25,511
    C,./tests/bench.c,60,10,482
    OCaml,./genfft/algsimp.ml,54,50,476
    C/C++ Header,./api/fftw3.h,25,72,428
    ```
1. Both of these outputs together should be used to identify hotspots. I used the `merge_comp_freqs.py` script to combine these results. For this analysis I am only interested in the top 20 listings. I used the command below to find them and store them into a file `hotspots.txt`.
    ```
    $ python /opt/ixmaat0.8.5/scripts0.4/merge_comp_freqs.py fftw3_freqs.csv fftw3_lines.csv | head -n 20 > hostpots.csv
    ```
    Shown below are the top twenty _potential_ hotspots which are worthy of further investigation.
    ```
    module,revisions,code
    configure.ac,17,686
    simd-support/simd-maskedsve.h,12,251
    CMakeLists.txt,11,378
    tests/fftw-bench.c,8,237
    simd-support/simd-common.h,6,73
    kernel/cycle.h,6,375
    kernel/ifftw.h,5,754
    simd-support/Makefile.am,5,17
    Makefile.am,4,155
    codemeta.json,4,55
    api/fftw3.h,4,428
    threads/threads.c,3,338
    m4/ax_cc_maxopt.m4,3,92
    simd-support/simd-avx2.h,3,272
    dft/codelet-dft.h,3,73
    genfft/util.ml,3,99
    genfft/annotate.ml,3,282
    dft/conf.c,3,85
    dft/simd/Makefile.am,3,4
    ```
    I know that this library is mostly written in C therefore I can reduce these hotspots further by only looking at source (.c) and header (.h) files. I used the grep utility and a regular expression to trim the results above. 
    ```
    $ grep -E '\.(h|c)\b' hotspots.txt | head -n 10
    ```
    Below are the final 10 identified hotspots that should be investigated further.
    ```
    simd-support/simd-maskedsve.h,12,251
    tests/fftw-bench.c,8,237
    simd-support/simd-common.h,6,73
    kernel/cycle.h,6,375
    kernel/ifftw.h,5,754
    api/fftw3.h,4,428
    threads/threads.c,3,338
    simd-support/simd-avx2.h,3,272
    dft/codelet-dft.h,3,73
    dft/conf.c,3,85
    ```

## 4. Hotspot Candidates
Interestingly, the top identified hotspot during analysis, `configure.ac`, should be excluded because it is only a build file and not actual source code. We should instead focus on these two files:
1. **simd-support/simd-maskedsve.h,12,251**
1. **tests/fftw-bench.c,8,237**

## 5. A Deeper Dive
Now that I've identified the top two files that are most likely hotspots I can inspect their measure of complexity by code shape. I used the `complexity_analysis.py` script on both files to determine an overall complexity number.

I fount that the file `simd-maskedsve.h` has a complexity of 307. Because this is a header file it makes sense that the maximum indentation is no more than 2.
```
n,total,mean,sd,max
307,87,0.28,0.48,2
```
The file `fftw-bench.c` has a complexity of 251 and its maximum indentation is only 3. For a C file this means it is likely well formatted and not that complex. 
```
n,total,mean,sd,max
251,170,0.68,0.6,3
```
⭐ The low complexity of these files does not suggest that they are indeed technical debt hotspots but let's analyze how this score evolved over time. 

I determined the first commit SHA in my timeframe by inspecting the HEAD of the fftw3_evo.log. I found it to be `7947c101`.

```
$ cat fftw3_evo.log | head -n 5
[7947c101] Matteo Frigo 2025-10-29 Get rid of the dependency on the Unix module
```
The last commit  can be found at the tail of the file: `504ece7f`.
```
$ cat fftw3_evo.log | tail -n 5

[504ece7f] Alexei Colin 2017-10-31 perf counters: add PMCCNTR for ARMv7 and add docs
93      0       README-perfcnt.md
8       3       configure.ac
12      0       kernel/cycle.h
```
Using these commit SHAs as bookends we can collect their complexity trends over the duration of the time span using these two commands:
```
$ python /opt/ixmaat0.8.5/scripts0.4/git_complexity_trend.py --start 504ece7f --end 7947c101 --file simd-support/simd-maskedsve.h
```
and 
```
$ python /opt/ixmaat0.8.5/scripts0.4/git_complexity_trend.py --start 504ece7f --end 7947c101 --file tests/fftw-bench.c
```
From these results I was able to plot a trend line for each file.  

The trend line plot for this file shown below indicates that the file was deteriorating at one point however, a single change increased the complexity significantly but ultimately provided stability. 

![simd-maskedsve.h](https://mailuc-my.sharepoint.com/:i:/r/personal/blohmjd_mail_uc_edu/Documents/Architecture/simd-maskedsve.jpg?csf=1&web=1&e=qX1cQX)

Conversely, this trend line indicates a recent refactor. At a moment in time the complexity decreased momentarily but this did not prevent complexity from rising still. 

![fftw-bench.c](https://mailuc-my.sharepoint.com/:i:/r/personal/blohmjd_mail_uc_edu/Documents/Architecture/fftw-bench.jpg?csf=1&web=1&e=A3a2t6)

⭐ I would not recommend these files for further inspection or refactoring effort. While changes to these files post release is an indicator of technical debt being paid, the relative complexities and change frequency of both files shows that the cost was is expensive.

## 6. A Sum of Coupling Analysis
Hotspots can also be found by tracking which files change together. Code that is too tightly coupled can indicate a bad design and should be refactored.  


The SCM log can be analyzed for a sum of changes with this command.
```
$ maat -l fftw3_evo.log -c git -a soc > fftw3_soc.csv
```
I will more the list to include (.c) and (.h) files and exclude (.am) files which are build files.
```
$ grep -E '\.(h|c)\b' fftw3_soc.csv | grep -v '\.am\b' | head -n 5
```
Here are the top results:
```
kernel/ifftw.h,218
rdft/conf.c,152
dft/conf.c,152
simd-support/simd-common.h,141
rdft/codelet-rdft.h,133
dft/codelet-dft.h,133
api/version.c,133
simd-support/simd-avx2.h,61
simd-support/avx2.c,57
kernel/scan.c,46
api/apiplan.c,44
```

## 7. A Temporal Analysis
Now that we have counted the SOC. We can again use maat to find a correlation between the files which were changed. 

❗I first performed this analysis and found no results. I expect that this is due to the timeframe that I selected. By picking a timeframe following major release it is reasonable to expect that the code was modified very precisely rather than in batches. To complete the assignment, I had to expand my timeframe by a few years to get results  (2014-2025).   

```
$ maat -l fftw3_evo.log -c git -a coupling > fftw3_coupling.csv
```
I will once more filter the list to include (.c) and (.h) files and exclude (.am) files.
```
$ grep -E '\.(h|c)\b' fftw3_coupling.csv | grep -v '\.am\b' | head -n 20
```
Here are the top results:
```
dft/conf.c,rdft/conf.c,100,10
dft/codelet-dft.h,rdft/codelet-rdft.h,100,8
dft/codelet-dft.h,rdft/conf.c,88,9
dft/conf.c,rdft/codelet-rdft.h,88,9
dft/codelet-dft.h,dft/conf.c,88,9
rdft/codelet-rdft.h,rdft/conf.c,88,9
api/version.c,rdft/conf.c,84,10
api/version.c,dft/conf.c,84,10
api/version.c,rdft/codelet-rdft.h,82,9
api/version.c,dft/codelet-dft.h,82,9
kernel/ifftw.h,rdft/conf.c,74,14
dft/conf.c,kernel/ifftw.h,74,14
api/version.c,simd-support/simd-common.h,66,11
kernel/ifftw.h,rdft/codelet-rdft.h,64,13
dft/codelet-dft.h,kernel/ifftw.h,64,13
dft/conf.c,simd-support/simd-common.h,63,11
rdft/conf.c,simd-support/simd-common.h,63,11
api/version.c,kernel/ifftw.h,61,13
rdft/codelet-rdft.h,simd-support/simd-common.h,60,10
dft/codelet-dft.h,simd-support/simd-common.h,60,10
```

The temporal couplding shown above indicates some strong coupling between implementation files and some _codelet_ files. Admittedly, I'm not sure what to make of that. This is an example where domain specific knowledge would be required to analyze further. 

## Contributions
I'm in the asynchronous section of the class and therefore I have not yet connected with classmates. I completed this assignment by myself. 

|Member|Contribution|
|-|-|
Joshua Blohm | 100%
