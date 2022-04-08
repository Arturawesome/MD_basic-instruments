
Autor: [Dmitryuk Nikita](https://github.com/NikitaDmitryuk)

Laboratory: [Soft Matter Group, Bauman MSTU](http://teratech.ru/en)

Github repository: [Lammps_compile_gpu_env](https://github.com/NikitaDmitryuk/Lammps_compile_gpu_env)

---

# Lammps nvidia-docker

---

***Docker*** позволяет создавать и запускать контейнеры со своей операционной системой и программами обособленно от компьютера, на котором запускается контейнер, соблюдая все зависимости и переменные среды. Это ускоряет развертывание программ на других компьютерах, так как не требуется устанавливать нужные зависимости и программы.


## Первоначальная настройка

Для работы контейнера необходиы ***драйвера nvidia***, ***docker*** и ***[nvidia-docker](https://github.com/NVIDIA/nvidia-docker)*** версии 2.5 или выше.

Перед использованием *docker* может понадобиться запустить службу:


```shell
sudo systemctl start docker
```

Необходимо загрузить следующий контейнер (вне зависимости от операционной системы на пк):

```shell
sudo docker pull nvidia/cuda:10.2-devel-ubuntu18.04
```

Следующие команды должны выполняться без ошибок:

```shell
docker run --runtime=nvidia --rm nvidia/cuda:10.2-devel-ubuntu18.04 nvidia-smi
sudo docker run --rm --runtime=nvidia nvidia/cuda:10.2-devel-ubuntu18.04 nvcc --version
```

## Сборка образа

В файле ***build_lammps_docker.sh*** необходимо указать имя контейнера (*NAME_CONTAINER*), а также версию [архитектуры видеокарты](https://ru.wikipedia.org/wiki/CUDA) (*ARCH_GPU*).
Дополнительные параметры компиляции можно изменить в ***Dockerfile***.

Папка с исходным кодом с названием **lammps** или **жесткая ссылка** на нее должны находиться в одной папке со скриптом и Dockerfile (*lammps* можно скачать командой: `git clone  --branch=stable https://github.com/lammps/lammps.git`).

Запустить скрипт:

```shell
sudo ./build_lammps_docker.sh
```

Это создаст контейнер и скомпилирует в нем lammps, который находится в папке ***/lammps в текущей директории***.

Собранный контейнер с названием *NAME_CONTAINER* хранится на компьютере, и не требует повторной сборки, его можно использовать из любой директории.

Список всех образов можно получить командой `sudo docker images`.

## Использование образа

Для запуска контейнера с установленным *lammps*:

```shell
nvidia-docker run --rm -ti -v $(pwd):/srv/input NAME_CONTAINER
```

Команда запускает контейнер с названием **NAME_CONTAINER**. Ключ *-v $(pwd):/srv/input* задает папку на компьютере, доступную контейнеру (путь до двоеточия), а в контейнере это будет путь, указанный после двоеточия.

В данном случае контейнеру будет доступна папка, из которой запускается контейнер, а в контейнере все файлы будут лежать в */srv/input*.

После запуска контейнера в нем можно выполнять команды и запускать lammps.

Пример:

```shell
lmp_g++_openmpi -sf gpu -pk gpu 1 -in in.file
```

Для выхода из контейнера можно использовать комбинацию клавиш `ctrl + d`.

---

# Lammps CMake build

---

Поместить файл ***build_lammps_cmake.sh*** в одну директорию с папкой *lammps*. В файле параметр ***gpu_arch*** отвечает за [архитектуру видеокарты](https://ru.wikipedia.org/wiki/CUDA).

Компиляция *lmp_g++_openmpi* выполняется командой:

```shell
bash build_lammps_cmake.sh
```

Исполняемый файл *lmp_g++_openmpi* появится в папе *build*. Символьная ссылка на него помещается в папку с моделированием.


---

# Hoomd-blue CMake build

---

***Hoomd-blue*** является библиотекой python. Для удобства использования нескольких разных скомпилированных библиотек используется [Anaconda3](https://docs.anaconda.com/anaconda/install/linux/).

## Dump в формате .lammpstrj

Для вывода dump-файлов в формате *.lammpstrj* необходимо поместить файлы *HOOMDDumpWriter.cc* и *HOOMDDumpWriter.h* находящиеся в папке *hoomd/deprecated* в аналогичную папку в *hoomd* с заменой, и отредактировать их для нужного вывода. 

После компиляции использовать команду: 

```python
dump.xml(group=hoomd.group.all(), filename="dump", period=100, type=True)
```

## Компиляция

В строчках в файле ***build_hoomd.sh*** необходимо указать имя окружения и путь до *Anaconda3* соответственно.

```shell
name_env="EnvName"
anaconda_path=$HOME/anaconda3
```

Запуск компиляции из папки с исходниками и скриптом:

```shell
bash build_hoomd.sh 2> log.txt
```

После компиляции библиотека автоматически устанавливается в окружение с названием *name_env="EnvName"*.

Для ее использования необходимо активировать нужную среду *Anaconda3*:

```shell
conda activate "name_env"
```

После этого библиотеку можно использовать импортитировав в python: `import hoomd as hd`


---

# Изменение потенциала взаимодействия

---

## Lammps

При расчете на видеокарте используется потенциал, находящийся в файле исходников *lammps* по адресу */lib/gpu/lal_lj.cu*.

Для его замены необходимо поменять строку в которой рассчитывается сила взаимодействия между частицами (желательно в обеих функциях (*void k_lj, void k_lj_fast*)):


```c++
...
numtyp force = r2inv*r6inv*(lj1[mtype].x*r6inv-lj1[mtype].y);
...
```

Сила взаимодействия рассчитывается как *force = - u' / r*, где *u* - функция потенциала взаимодействия.

После чего необходимо заново скомпилировать *lammps* (пункт [сборки образа](https://github.com/NikitaDmitryuk/Lammps_compile_gpu_env/blob/main/README.md#%D1%81%D0%B1%D0%BE%D1%80%D0%BA%D0%B0-%D0%BE%D0%B1%D1%80%D0%B0%D0%B7%D0%B0)).

Новому образу можно дать название, соответствующее потенциалу взаимодействия (*NAME_CONTAINER*).

***

Для использования этого потенциала в моделировании *in.file* должен содержать следующие строки:

    ...
    package gpu 1
    ...
    pair_style lj/cut ${Rc}
    ...

*Lammps* необходимо запускать с ключами `-sf gpu -pk gpu 1`.

## Hoomd-blue

Исходники [потенциалов](https://hoomd-blue.readthedocs.io/en/v2.9.7/module-md-pair.html) Hoomd находятся в папке *\hoomd\md* c названиями ***EvaluatorPair\*.h***.

Все числовые типы данных должны быть заданы как ***Scalar(число)***.
