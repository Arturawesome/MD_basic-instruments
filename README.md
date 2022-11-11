
Autor: [Artur D. Nasyrov](https://github.com/Arturawesome)

Laboratory: [Bauman Digital Soft Matter laboratory, BMSTU](http://teratech.ru/en)

Operating System: Manjaro Linux KDE Plasma Version: 5.22.5. 

Processors: 8 × Intel® Core™ i7-9700KF CPU @ 3.60GHz

---

# VMD install

---

***VMD*** представляет собой удобный инстурмент для визуализации молекулярной динамики. 


Для установки  ***VMD***, его необходимо скачать с ***[сайта разработчика](https://www.ks.uiuc.edu/Research/vmd/)***. Далее создать папку в которой будет установлена программа и поместить туда скачаный архив (Для примера разархивация происходит в папке *2021*). Далее необходимо выполнить следующие команды.


```shell
[artur@system 2021]$ tar -xvf vmd   
[artur@system 2021]$ cd vmd1.9.3
[artur@system vmd1.9.3]$ ./configure
[artur@system vmd1.9.3]$ cd src
[artur@system src]$ sudo make install
```

Запуск производится командой

```shell
vmd
```


# Lammps install

***[Lammps](https://www.lammps.org/)***  - пакет созданный для молекулярной динамики на основе языков ***C++*** и ***CUDA*** . 


Необходимо скачать и разархивировать Lammps архив. (Для примера разархивация происходит в папке *2021*)

```shell
[artur@system 2021]$  tar xvf lammps.tar.gz
```
В результате образуется папка типа ***lammps-29Sep2021***. В папке 2021 создаем bash файл ***install.sh*** файл с содержанием
```shell
cmake -B build_LJ -S lammps-29Sep2021/cmake
make clean
cmake build_LJ\
    -D CMAKE_INSTALL_PREFIX=/opt\
    -D CMAKE_LIBRARY_PATH=/opt/cuda/lib64/stubs\
    -D BIN2C=/opt/cuda/bin/bin2c\
    -D LAMMPS_MACHINE=g++_openmpi_LJ\
    -D PKG_GPU=on\
    -D GPU_API=cuda\
    -D PKG_ASPHERE=on\
    -D PKG_KSPACE=on\
    -D PKG_MANYBODY=on\
    -D PKG_MOLECULE=on\
    -D PKG_BROWNIAN=on\
    -D PKG_RIGID=on\
    -D PKG_MISC=on\

    cmake --build build_LJ -j4
```
Далее запускаем данный bash файл. По жулаению и надобности можно редактировать содержание включенных пакетов. 
```shell
[artur@system 2021]$ bash install.sh
```

В результате получаем папку ***build_LJ*** с  исполняемый файлом ***g++_openmpi_LJ***  внутри. Для удобвства можно создать сивольную ссылку на исполнительный файл.  Для зпуская скрипта использовать команду. 

```shell
g++_openmpi_LJ -sf gpu -pk gpu 1 -in input.script
```



---

# HOOMD-blue build

***[Hoomd-blue](http://glotzerlab.engin.umich.edu/hoomd-blue/)*** является библиотекой для *python*, для молекулярной динамики, которая изначально создавалась для расчетов на видеокарте.
## Pre-install
Для работы ***HOOMD-blue*** необходимы следующие пакеты:
1. Python >= 3.5           (in majaro Linux should be initially)
2. NumPy >= 1.7
3. CMake >= 2.8.10.1
4. Make
5. gcc >= 4.8
6. pybind11 >= 2.2
7. Eigen >= 3.2
8.  nvidia cuda toolkit
9.  openmpi                 (in majaro Linux should be initially)

```shell
[artur@system ]$ sudo pacman -S name_environment
```

## Pre-install. Dump trajectory file
***HOOMD-blue*** имеет неудобные выходные файлы траекторий, которые нельзя использовать в ***VMD***. Для вывода dump-файлов в формате *.lammpstrj* необходимо поместить файлы *HOOMDDumpWriter.cc* и *HOOMDDumpWriter.h*  в папку *hoomd/deprecated* с заменой.

## Install
Вопреки инструкциям с оффициального сайта ***[HOOMD-blue](https://hoomd-blue.readthedocs.io/en/v3.0.0/building.html)*** создавать и активировать среду *python* в отдельной папке не нужно.
Наиболее удобным в данном случае будет установка ***HOOMD-blue***  без использования *bash* файла.  Для примера работа происходит в папке *2021*. Необходимо скачать и разархивировать HOOMD архив.

```shell
[artur@system 2021]$ curl -O https://glotzerlab.engin.umich.edu/Downloads/hoomd/hoomd-v2.9.7.tar.gz
[artur@system 2021]$ tar xvzf hoomd-v2.9.7.tar.gz
```
Войти в папку с исходниками и создать папку *build* и перейти в нее

```shell
[artur@system 2021]$ cd hoomd-v2.9.7
[artur@system hoomd-v2.9.7]$ mkdir build
[artur@system hoomd-v2.9.7]$ cd build
```
в папке *build* выполнить команды:
```shell
[artur@system build]$ cmake ../
[artur@system build]$ ccmake ../
```
В спывшем окне поменять строчку с названием *CMAKE_INSTALL_PREFIX* на желаемый путь установки пакета HOOMD, активировать нужные вам параметры компиляции. И сохранить изменения. После редакции в ccmake выполнить компиляцию HOOMD:
```shell
[artur@system build]$ make -j4
[artur@system build]$ ctest
```
выполнить установку 
```shell
[artur@system build]$ make install -j4
```
 Компиляция *HOOMD* выполняется один раз, если вы поменяли исодники (например поменяли потенциал взаимодействия частиц) то выполнять надо будет тольк *$ make install -j4*. Предыдушие шаги выполнять не надо


---

# Изменение потенциала взаимодействия LAMMPS, HOOMD-blue

---

## Lammps

При расчете на видеокарте используется потенциал *LJ12-6*, находящийся в файле исходников *lammps* по адресу */lib/gpu/lal_lj.cu*.

Для его замены необходимо поменять строку в которой рассчитывается сила взаимодействия между частицами (желательно в обеих функциях (*void k_lj, void k_lj_fast*)):


```c++
...
numtyp force = r2inv*r6inv*(lj1[mtype].x*r6inv-lj1[mtype].y);
...
```


После чего необходимо заново скомпилировать *lammps*.


***

Для использования этого потенциала в моделировании *input.script* должен содержать следующие строки:

    ...
    package gpu 1
    ...
    pair_style lj/cut ${Rc}
    ...

*Lammps* необходимо запускать с ключами `-sf gpu -pk gpu 1`.

## Hoomd-blue

Исходники [потенциалов](https://hoomd-blue.readthedocs.io/en/v2.9.7/module-md-pair.html) Hoomd находятся в папке *\hoomd\md* c названиями ***EvaluatorPair\*.h***.

Для добавления каких-либо сторонних сил, непредусмотренных в исходниках *HOOMD* удобно работать в файлах *TwoStep_TYPEmd_.cu*. Расчет в данных файлах ведется относительно ОТДЕЛЬНО взятой частицы (не массив в целом).

---

# Загрузка и компиляция покета вороного

---
Загрузить pkbbuild файл из ***AUR*** и заменить строчку загрузки с сайте LAMMPS на офф.зайт пакета. Пример правильного файла:
```shell
# Maintainer: xpt <user.xpt@gmail.com>
pkgname=voro++
pkgver=0.4.6
pkgrel=2
pkgdesc="A free software library for the computation of three dimensional Voronoi cells."
arch=('i686' 'x86_64')
url="http://math.lbl.gov/voro++/"
license=('BSD')
depends=('perl' 'gcc-libs')
options=(staticlibs)
source=(https://math.lbl.gov/voro++/download/dir/voro++-0.4.6.tar.gz)
md5sums=('2338b824c3b7b25590e18e8df5d68af9')
 
build() {
  cd "$srcdir/$pkgname-$pkgver"
  sed -i 's/(PREFIX)\/man/(PREFIX)\/share\/man/g' Makefile
  sed -i 's/-Wall/-Wall -fPIC/g' config.mk
  make all
}

package() {
  cd "$srcdir/$pkgname-$pkgver"
  make PREFIX=$pkgdir/usr install
  install -d $pkgdir/usr/share/licenses/$pkgname $pkgdir/usr/share/$pkgname/{examples,scripts} $pkgdir/usr/share/doc/$pkgname/{,html}
  install -m644 $srcdir/$pkgname-$pkgver/LICENSE $pkgdir/usr/share/licenses/$pkgname
  install -m644 $srcdir/$pkgname-$pkgver/NEWS $pkgdir/usr/share/doc/$pkgname
  install -m644 $srcdir/$pkgname-$pkgver/scripts/* $pkgdir/usr/share/$pkgname/scripts
  install -m644 $srcdir/$pkgname-$pkgver/html/* $pkgdir/usr/share/doc/$pkgname/html
  cp -r $srcdir/$pkgname-$pkgver/examples/* $pkgdir/usr/share/$pkgname/examples
  chmod -R 755 $pkgdir/usr/share/$pkgname/examples
} 
```
Запуск делать при помощи команды 
```shell
makepkg -is   
```
