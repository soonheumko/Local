---
layout: menu
title: Fortran I/O Performance
description: Suggestions for improving the I/O performance in your Fortran code
published: true
---

# Improving I/O performance in your Fortran code

Many users report the bad I/O performance especially when they treat much larger problem size than before. Sometimes some users doubt the performance of GPFS file system which builds our centre storage system. However, in most cases, the poor I/O performance inherits from the bad programming in your code. General guidance in this article We address the general guidance in this article, hoping that it contributes to improving the I/O performance of your Fortran code.

We intend to provide an advice to users who use Intel Fortran compiler. Thus, some tips ([Section 3](#sec3) and [Section 4](#sec4)) may not apply to the GNU compiler users. Also, this article covers the general tips for the sequential I/O operation (not MPI-I/O). Finally, we would remark that many of the contents are referenced by an online article [here](http://kiwi.atmos.colostate.edu/rr/tidbits/intel/macintel/doc_files/source/extfile/optaps_for/fortran/optaps_prg_io_f.htm).

## 1. Use of Unformatted File

By default, Fortran treats data files as FORMATTED (so-called ASCII format or plain text). The formatted file has a strong advantage that you can easily handle the dataset by any text editor. However, the formatted file spends longer time for I/O operation than unformatted one due to the need of converting the native unformatted data to character strings. At the same time, the file size becomes larger than the unformatted file and you may lost precision when the data is read back to another code. Therefore, we would recommend you to **handle your data in unformatted format**.

To save/load your data in unformatted format, add UNFORMATTED *form* specifier in your OPEN statement as follows:

``` fortran
	open(10,file='datafile',form='UNFORMATTED')
```

One remark is that the native unformatted data is not portable by 100 percent. In other words, the format may differ depending on the operating system and the compiler. The common issue of different endianness (big-endian VS little-endian) between compilers can be controlled by the proper compilation option or code change. Instructions for the Intel compiler can be found [here](http://kiwi.atmos.colostate.edu/rr/tidbits/intel/macintel/doc_files/source/extfile/optaps_for/fortran/optaps_cmp_endi_f.htm).

<a name="sec2">
## 2. Intermediate storage on RAM and Writing the array instead of single element
</a>

Access to the peripheral disk for reading/writing operation is much slower than copying the data between separate RAM memory space. At the same time, the disk I/O speed is highly dependent on the number of instructions to access the file. As an example,

<a name="example_1">
* Example 1
</a>
``` fortran
	integer, parameter :: N=10000000 ! sufficiently large number
	integer :: arr(N), i
	open(10,file='dummy',form='UNFORMATTED')
	! Case 1: Good example of performing I/O operation
	forall (i=1:N) arr(i)=i
	write(10) arr
	! Case 1-1: A more general version of Case 1
	write(10) (arr(i),i=1,N)
	! Case 2: Bad example of performing I/O operation
	write(10) (i,i=1,N)
	! Case 3: Horribly slow example of performing I/O operation
	do i=1,N
	 write(10) i
	enddo
	! End of cases
	close(10)
```

the I/O speed in the first case is noticeably faster than the second case. The third case of writing the record one-by-one is horribly slower than others.

To explain in detail, the compiler transfers the block of neighbouring entries from an array (*arr(i)* in [Example 1](#example_1)) to the device in *Case 1**. Therefore, multiple elements in an array are written to the disk at a single instruction. In *Case 2*, the compiler first stores the value of a single variable (*i* in [Example 1](#example_1)) in an internal buffer and passes the buffer array to the device for writing operation. In *Case 3*, a single variable is passed to the device for performing the writing instruction. Therefore, much more instruction is necessary to complete the writing operation than others, who transfer a pack of dataset at an instruction.

We recommend you to have the habit of **writing the array as a bulk and avoiding the I/O statement within DO-loop**.

<a name="sec3">
## 3. Effective use of I/O buffer (Intel Compiler)
</a>

Intel Fortran run-time library is said to provide the disk block I/O buffer, where multiple I/O streams are packed together and transferred to the physical disk I/O for operation. By default, the Fortran run-time system conducts the unbuffered disk writes.

It contributes to improving the writing performance, especially if the size of element at a single instruction is small (the *Case 3* in [Example 1](#example_1)). We also observe that the writing performance is significantly improved if your file is of FORMATTED one. Meanwhile, the disk block I/O buffer consumes local RAM space, which can result in the OOM (Out-Of-Memory) situation depending on your data size.

To enable the buffered write, you can choose one of following ways:

* Open each file with buffered specifier so that **OPEN(unit,file=name,BUFFERED='YES')**
* Compile with **-assume buffered_io** flag
* Type **export FORT_BUFFERED=true** in the command line

The BUFFERED statement contributes to opening a single file with the disk block buffer assigned and it precedes over other global instructions of compilation flag or run-time environment. Like similar situations, compilation flag precedes the run-time configuration.

**The use of buffered I/O is generally recommended for improving the writing performance, but be careful not to consume too much RAM**.

<a name="sec4">
## 4. Reading a single file from multiple MPI ranks (Intel Compiler version 13 and above)
</a>

From Intel compiler 13 and onward versions, we observe that file reading speed drastically decreases at certain cases. As noted in this [online discussion](https://software.intel.com/en-us/forums/topic/549671), it slows down if **multiple cores access a single file to read the snippet** of it and the code **mixes single and array variables to scan the line of a datafile**. If your read statement is implemented in this manner, you are most likely to encounter the same slowdown we observed:

``` fortran
	read(unit) ((tmp,i=1,start-1),(arr(i),i=start,end),(tmp,i=end,max))
```

where *tmp* is a dummy variable which dumps away unnecessary data, *arr* is the actual array which stores the valid dataset, and *start*,*end*,*max* refers to the range of a dataset.

The reading performance can be improved by changing a program or by turning on the buffered I/O which is explained above. With regard to the change on program, the performance is much improved if the dummy variable is designed as an array instead of a single variable:

``` fortran
	read(unit) ((tmp1(i),i=1,start-1),(arr(i),i=start,end),(tmp2(i),i=end,max))
```

Beware that we now allocate dummy arrays of ***tmp1(:)*** and ***tmp2(:)*** to store unnecessary dataset from the data file, instead of keep overwriting one ***tmp*** variable.

To revisit a way of turning on buffered I/O,

<p>
	**OPEN(unit,file=name,BUFFERED='YES')** or **-assume buffered_io** or **export FORT_BUFFERED=true**
</p>

will make use of disk block I/O buffer.

At least we clearly observe the above slowdown when **more than 100 CPU cores access a single datafile of bigger than 100 Megabytes** to read a part of the data file. If your case applies to this range, we highly recommend you to **turn on buffered I/O feature and rerun the same simulation** to see the performance change.


## 5. Results

* Verification of Section 1

<ul><ul>We first compare the performance between FORMATTED and UNFORMATTED file reading/writing performances. We compare the performance of writing and reading a 1-dimensional integer array whose size is 100 million (roughly 380 Megabytes in file size). We write/read the entire variable at a single instruction. It is tested on Intel compiler version 15. We did not apply the buffered I/O option.<br><br></ul>


| I/O Instruction | File Format | Time for Operation | 
|:-----:|:-----:|:-----:|
| READ | FORMATTED | 13.1749629974365 |
|  | UNFORMATTED | 9.195303916931152E-002 |
| WRITE | FORMATTED | 77.7938079833984 |
|  | UNFORMATTED | 0.879778146743774 |

<br> This result tells you that I/O speed accelerates by roughly 100 times with UNFORMATTED file format. The result can differ by the data size and the overhead in I/O server.
</ul>

* Verification of Section 2

<ul>
We next compare the performance between completing I/O instruction by a single function call and reading/writing a single entry under the DO-loop ([Example 1](#example_1) in [Section 2](#sec2)).

* UNFORMATTED Dataset

| I/O Instruction | Function Call | Time for Operation | 
|:-----:|:-----:|:-----:|
| READ | Single Call | 9.195303916931152E-002 |
|  | Call within DO-Loop | 10.6995301246643 |
| WRITE | Single Call | 0.879778146743774 |
|  | Call within DO-Loop | 467.021706104279 |

* FORMATTED Dataset

| I/O Instruction | Function Call | Time for Operation | 
|:-----:|:-----:|:-----:|
| READ | Single Call | 13.1749629974365 |
|  | Call within DO-Loop | 27.1288828849792 |
| WRITE | Single Call | 77.7938079833984 |
|  | Call within DO-Loop | 522.829308032990 |

Worth to note that completing the I/O operation by a single call provides much better performance than the call within a loop. The gap is bigger if the file is in UNFORMATTED format.
</ul>

* Verification of Section 3
<ul>
The effect of buffered I/O is measured by activating and deactivating the FORT_BUFFERED variable. We only demonstrate the writing performance, since we did not see the noticeable change on read performance.

* UNFORMATTED Dataset

| Function Call | Buffered I/O | Time for Operation | 
|:-----:|:-----:|:-----:|
| Single Call | TRUE | 0.491503000259399 |
|  | FALSE | 0.879778146743774 |
| Call within DO-Loop | TRUE | 13.4019091129303 |
|  | FALSE | 467.021706104279 |

* FORMATTED Dataset

| Function Call | Buffered I/O | Time for Operation | 
|:-----:|:-----:|:-----:|
| Single Call | TRUE | 7.28528094291687 |
|  | FALSE | 77.7938079833984 |
| Call within DO-Loop | TRUE | 22.0161950588226 |
|  | FALSE | 522.829308032990 |

As we expected, turning on buffered I/O affects much if the size of writing data is small. Interestingly, the use of disk block buffer also contributes to writing the entire dataset through a single instruction.
</ul>

* Verification of Section 4
<ul>
The last experiment measures the read performance when multiple cores access a single data file. The data file consists of 20 million double-precision values. The dataset is equally partitioned to 256 blocks and each core acceses the same data file to read its own partition. We compare performances between setting the dummy variable (which reads the data in other partition) as a single variable and the array of actual size. The data format is an UNFORMATTED one.<br><br>

| Dummy Variable | Buffered I/O | Time for Operation | 
|:-----:|:-----:|:-----:|
| Single | TRUE | 0.963283278979361 |
|  | FALSE | 732.049077565782 |
| Array | TRUE | 0.221010014414787 |
|  | FALSE | 0.143553297035396 |

We recognize that the performance is highly improved by the buffered I/O if the dummy variable is of size 1. No further gain is achieved if the dummy variable is designed as the array. Though not listed here, the performance did not differ if the read operation is called within a DO-loop. In those cases, the read operation took in the range of 3.2 to 3.4 seconds.
</ul>

## 6. Summary

As the short summary,

<strong><ul>
- If you do not migrate among different sites frequently, use of UNFORMATTION file formulation can contribute much on improving I/O performance.
- Try to call least number of I/O functions within your code. That is, try to pack multiple data in a contiguous memory space and read/write multiple data at once.
- For the Intel compiler user, turning on buffered I/O option will give you the improved writing performance in general. This also can be a solution to the ones who experience the sudden slowdown of file read performance after the migration to higher Intel compiler version.
</ul></strong>
