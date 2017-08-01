---
title: cfl3d+openmpi+gfortran编译和安装
---

# 简介

cfl3d用intel的编译器和mpi会比较容易编译，不再详述。本文主要讲述用gfortran的情况

cfl3d依赖于cgns, fortran编译器和mpi

- mpi：只要是符合mpi1.1的标准的MPI应该都可以的,mpich, openmpi, intelmpi都可以
- fortran编译器：gfortran, ifort都可以；
- cgns库：不是必须的，但是有个算例会用到，用apt-get安装的cgns库在链接的时候会找不到hdf5库。
  - 所以根据cfl3d说明和网上的资料，本文选择cgns 2.5.5来进行自主编译

# 下载和解压cfl3d 6.7 开源版本

```shell
cd $HOME
mkdir cfl3d
cd cfl3d
git clone https://github.com/nasa/CFL3D.git
```

这样就在`$HOME/cfl3d/CFL3D`中下载到了所有源代码。

# MPI和gfortran

在ubuntu下

```shell
sudo apt-get install libopenmpi-dev gfortran build-essential -y
```

在RHEL/Centos下

```shell
sudo yum install openmpi-devel gcc gcc-gfortran
```

# 下载和安装cgns

```shell
cd $HOME/cfl3d
wget "http://jaist.dl.sourceforge.net/project/cgns/cgnslib_2.5/Release 5/cgnslib_2.5-5.tar.gz" -O cgnslib_2.5-5.tar.gz
tar xf cgnslib_2.5-5.tar.gz
mkdir cgns
mkdir cgns/include
mkdir cgns/lib
cd cgnslib_2.5-5
./configure --prefix=../cgns #这是按照最简单的配置进行编译的，没有配hdf5，并行之类高级的东西。
make
make install
```

# 生成cfl3d的makefile文件

运行`build/Install`

```shell
cd $HOME/cfl3d/CFL3D/build
./Install -fastio -cgnsdir=$HOME/cfl3d/cgns
```

之后会生成`build/makefile`文件可按下面进行更改。

```
# makefile
FFLAG        = -O3 -fbacktrace -w -march=native -J../../$(CFLLIBSD)
FFLAG_SPEC   = -O3 -fbacktrace -w -march=native -J../../$(CFLLIBSD)
PREC         = -fdefault-real-8
PREC_MPI     = -DDBLE_PRECSN
LFLAG        =
## 没有libcommon.a在编译precfl3d时会出现undefined reference 错误。
LLIBS        = -L/home/di/cfl3d/cgns/lib -lcgns ../../$(CFLLIBSD)/libcommon.a
LLIBS_SEQ    = -L/home/di/cfl3d/cgns/lib -lcgns ../../$(CFLLIBSD)/libcommon.a
## 因为后面用了mpif90作为fortran编译器和mpicc作为c编译器，所以MPI_INCDIR设置为空
MPI_INCDIR   =
CGNS_INCDIR  = -I/home/di/cfl3d/cgns/include
CPP          = cpp
CPPFLAG      = -P
CPPOPT       = -DFASTIO $(MPI_INCDIR) -DDIST_MPI $(PREC_MPI)
CPPOPT_SP    = -DP3D_SINGLE -DLINUX -DCGNS $(PREC_MPI) $(CGNS_INCDIR)
CPPOPT_CMPLX = -DCMPLX
FTN          = mpif90
CC           = mpicc
CFLAG        =
AROPT        = rusc
RANLIB       = true
INLINE       =
```

说明：

- `MPI_INCDIR`为空，因为`FTN`和`CC`编译器已经设置为`mpi`包装过的版本了。
- `CGNS_INCDIR`似乎必须设置为绝对路径；
- `AROPT`设置为`rusc`会报警告，可以无视，也可以改成`cr`啥的。
- 原文件选项的意义：
  - `-z muldefs`: 等价于`--allow-multiple-definition`，让编译器遇到多个定义时，用第一个而不是报致命错；

# 最终编译

```shell
make cfl3d_seq cfl3d_mpi splitter cfl3d_tools
make precfl3d
make ronnie preronnie
make maggie
make splittercmplx cfl3dcmplx_seq cfl3dcmplx_mpi 
```

## 代码修正

### 复数版本的错误

cfl3dcmplx_seq和cfl3dcmplx_mpi会因为cfl3dcmplx_libs编译不成功（spalart.F的turre的类型成了complex，不能比较大小）而失败。其他的都可以编译成功。

查看问题所在

```shell
di@ubuntu:~/cfl3d/CFL3D/build$ grep -n "turre.*.lt.*" cfl/libs/*.Fdi@ubuntu:~/cfl3d/CFL3D/build$ grep -n "turre.*.lt.*" cfl/libs/*.F
cfl/libs/barth3d.F:1863:              if((real(turre(j,k,i)+vist3d(j,k,i))) .lt. 1.e-12) then
cfl/libs/foureqn.F:1876:              if (real(turre(j,k,i,4)) .lt. 400.) then
cfl/libs/foureqn.F:1880:     +                 real(turre(j,k,i,4)) .lt. 596.) then
cfl/libs/foureqn.F:1885:     +                 real(turre(j,k,i,4)) .lt. 1200.) then
cfl/libs/global.F:3818:     .         ' automatically to turre...plt'
cfl/libs/global.F:3833:     .         ' automatically to turre...plt'
cfl/libs/spalart.F:547:              if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:611:              if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:674:              if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:787:              if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:850:              if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:912:              if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:1026:                if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:1092:                if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:1157:                if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:1297:              rd = turre(j,k,i)/(sqrt(velterm)*akarman*akarman*
cfl/libs/spalart.F:1414:              if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:1480:              if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:1643:              if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:1738:              if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:1832:              if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:1945:              if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:2029:              if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:2112:              if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:2215:                if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:2301:                if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:2386:                if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/spalart.F:2463:     +          (real(turre(j,k,i)+vist3d(j,k,i))) .lt. 1.e-12) then
cfl/libs/spalart.F:2468:     +          (real(turre(j,k,i)+vist3d(j,k,i))) .lt. 0.) then
cfl/libs/spalart.F:2503:            if (i_saneg .eq. 1 .and. turre(j,k,i) .lt. 0.) then
cfl/libs/twoeqn.F:3344:              d5 = re*beta5*ft*turre(j,k,i,1)**1.5/(sqrt(rt)+delta)
```

说明没有加real是主要的问题！

解决方案：turre(j,k,i)修改为了real(turre(j,k,i))

```shell
sed -i -e "s/\sturre(j,k,i)\s/ real(turre(j,k,i))/g" ../source/cfl3d/libs/spalart.F
sed -i -e "s/sbar \.ge\. -0\.7\*ss/real(sbar) .ge. -0.7*real(ss)/g" ../source/cfl3d/libs/spalart.F
```

sed所用的正则表达式可以在http://regexr.com/ 编辑和测试得到。

### maggie错误

maggie有位于`cputim.F`关于`ETIME`函数的错误，这是因为gfortran的ETIME和其他编译器不同，多带了一个参数。利用gfortran自带定义的`__GFORTRAN__`宏可以进行条件编译。

```fortran
! cputim.F :109
!#if defined ETFLAG
!      call etime_(tm)
!      call itime_(ia)
!#else
!      call etime(tm)
!      call itime(ia)
!#endif
#if defined ETFLAG
      call etime_(tm)
      call itime_(ia)
#elif defined __GFORTRAN__
      call etime(tm(1:2),tm(3))
      call itime(ia)
#else
      call etime(tm)
      call itime(ia)
#endif
```

# 设置链接

```shell
cd $HOME/cfl3d/CFL3D
mkdir bin
cd bin
## basic
ln -s ../build/cfl/seq/cfl3d_seq
## parallel version
ln -s ../build/cfl/mpi/cfl3d_mpi
## mesh block splitter
ln -s ../build/split/seq/splitter
## memory usage estimator
ln -s ../build/precfl/seq/precfl3d
## overset mesh related tool
ln -s ../build/mag/seq/maggie
## mesh deformation tool and its memory usage estimator
ln -s ../build/ron/seq/ronnie
ln -s ../build/preron/seq/preronnie
## complex version of cfl3d which is used to compulate flight derivative
ln -s ../build/cflcmplx/seq/cfl3dcmplx_seq
ln -s ../build/cflcmplx/mpi/cfl3dcmplx_mpi
ln -s ../build/splitcmplx/seq/splittercmplx
## cfl3d_tools
ln -s ../build/tools/seq/v6inpdoubhalf
ln -s ../build/tools/seq/Get_FD
ln -s ../build/tools/seq/initialize_field
ln -s ../build/tools/seq/nmf_to_cfl3dinput
ln -s ../build/tools/seq/cgns_to_cfl3dinput
ln -s ../build/tools/seq/v6inpswitchijk
ln -s ../build/tools/seq/v6_ronnie_mod
ln -s ../build/tools/seq/moovmaker
ln -s ../build/tools/seq/grid_perturb_cmplx
ln -s ../build/tools/seq/v6_restart_mod
ln -s ../build/tools/seq/gridswitchijk
ln -s ../build/tools/seq/cfl3d_to_nmf
ln -s ../build/tools/seq/cgns_readhist
ln -s ../build/tools/seq/p3d_to_cfl3drst
ln -s ../build/tools/seq/everyother_xyz
ln -s ../build/tools/seq/p3d_to_INGRID
ln -s ../build/tools/seq/v6_ronnie_mod.F90
ln -s ../build/tools/seq/INGRID_to_p3d
ln -s ../build/tools/seq/grid_perturb
ln -s ../build/tools/seq/cfl3dinp_to_FVBND
ln -s ../build/tools/seq/cfl3d_to_pegbc
ln -s ../build/tools/seq/plot3dg_to_cgns
ln -s ../build/tools/seq/XINTOUT_to_ovrlp
```

# 测试算例

测试算例的地址在：

https://cfl3d.larc.nasa.gov/Cfl3dv6/cfl3dv6_testcases.html

可以通过如下脚本下载所有测试算例

```shell
cd $HOME/cfl3d
mkdir TestCases
cd TestCases
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/Flatplate/Flatplate.tar.Z
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/Flatplateskew/Flatplateskew.tar.Z
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/Flatplateyplus/Flatplateyplus.tar.gz
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/Backstep/Backstep.tar.gz
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/Transdiff/Transdiff.tar.Z
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/NACA_4412/NACA_4412.tar.Z
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/RAE_Sensitivity/RAE_Sensitivity.tar.gz
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/Ramp/Ramp.tar.Z
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/Cylinder/Timeaccstudy.tar.Z
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/N0012/Spaceaccstudy.tar.Z
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/Ejectornozzle/Ejectornozzle.tar.gz
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/Pitch/Pitch0012.tar.Z
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/Rotorstator/Rotorstator.tar.Z
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/Hump/Humpcase.tar.gz
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/2DTestcases/Curvature/SoMellor.tar.gz
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/3DTestcases/Axibump/Axibump.tar.gz
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/3DTestcases/ONERA_M6/ONERA_M6.tar.Z
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/3DTestcases/ARA_M100/ARA_M100.tar.Z
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/3DTestcases/ARA_M100_XMERA/ARA_M100_XMERA.tar.Z
wget https://cfl3d.larc.nasa.gov/Cfl3dv6/3DTestcases/Delta/Delta_cgns.tar.Z
```

# 总结

上述修正已经提交到我的https://github.com/chengdi123000/CFL3D了，所以可以用以下脚本安装所有东西。

```shell
# MPI和fortran，UBUNTU为例
sudo apt-get update
sudo apt-get install libopenmpi-dev gfortran build-essential git -y
# 下载chengdi123000修正过的cfl3d代码，主要是spalart.F和cputim.F
cd $HOME
mkdir cfl3d
cd cfl3d
git clone https://github.com/chengdi123000/CFL3D.git
# cgns代码和安装
cd $HOME/cfl3d
wget "http://jaist.dl.sourceforge.net/project/cgns/cgnslib_2.5/Release 5/cgnslib_2.5-5.tar.gz" -O cgnslib_2.5-5.tar.gz
tar xf cgnslib_2.5-5.tar.gz
mkdir cgns
mkdir cgns/include
mkdir cgns/lib
cd cgnslib_2.5-5
./configure --prefix=../cgns #这是按照最简单的配置进行编译的，没有配hdf5，并行之类高级的东西。
make
make install
# 生成makefile
cd $HOME/cfl3d/CFL3D/build
./Install -fastio -cgnsdir=$HOME/cfl3d/cgns
# 编译cfl3d
cd $HOME/cfl3d/CFL3D/build
cp makefile_linux_gfortran_openmpi makefile #覆盖掉新生成的makefile
make cfl3d_seq cfl3d_mpi splitter cfl3d_tools
make precfl3d
make ronnie preronnie
make maggie
make splittercmplx cfl3dcmplx_seq cfl3dcmplx_mpi 
# 建立链接
cd $HOME/cfl3d/CFL3D
mkdir bin
cd bin
## basic
ln -s ../build/cfl/seq/cfl3d_seq
## parallel version
ln -s ../build/cfl/mpi/cfl3d_mpi
## mesh block splitter
ln -s ../build/split/seq/splitter
## memory usage estimator
ln -s ../build/precfl/seq/precfl3d
## overset mesh related tool
ln -s ../build/mag/seq/maggie
## mesh deformation tool and its memory usage estimator
ln -s ../build/ron/seq/ronnie
ln -s ../build/preron/seq/preronnie
## complex version of cfl3d which is used to compulate flight derivative
ln -s ../build/cflcmplx/seq/cfl3dcmplx_seq
ln -s ../build/cflcmplx/mpi/cfl3dcmplx_mpi
ln -s ../build/splitcmplx/seq/splittercmplx
## cfl3d_tools
ln -s ../build/tools/seq/v6inpdoubhalf
ln -s ../build/tools/seq/Get_FD
ln -s ../build/tools/seq/initialize_field
ln -s ../build/tools/seq/nmf_to_cfl3dinput
ln -s ../build/tools/seq/cgns_to_cfl3dinput
ln -s ../build/tools/seq/v6inpswitchijk
ln -s ../build/tools/seq/v6_ronnie_mod
ln -s ../build/tools/seq/moovmaker
ln -s ../build/tools/seq/grid_perturb_cmplx
ln -s ../build/tools/seq/v6_restart_mod
ln -s ../build/tools/seq/gridswitchijk
ln -s ../build/tools/seq/cfl3d_to_nmf
ln -s ../build/tools/seq/cgns_readhist
ln -s ../build/tools/seq/p3d_to_cfl3drst
ln -s ../build/tools/seq/everyother_xyz
ln -s ../build/tools/seq/p3d_to_INGRID
ln -s ../build/tools/seq/v6_ronnie_mod.F90
ln -s ../build/tools/seq/INGRID_to_p3d
ln -s ../build/tools/seq/grid_perturb
ln -s ../build/tools/seq/cfl3dinp_to_FVBND
ln -s ../build/tools/seq/cfl3d_to_pegbc
ln -s ../build/tools/seq/plot3dg_to_cgns
ln -s ../build/tools/seq/XINTOUT_to_ovrlp
# 下载算例
cd $HOME/cfl3d/testcase
tar xf Cfl3dv6.tar.gz
```

