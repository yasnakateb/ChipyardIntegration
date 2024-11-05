# Chipyard Integration
 

## Integrating a RoCC Accelerator with Chipyard (a.k.a., the ultimate guide for the brave)

In this guide, I’ll take you through the (sometimes frustrating) journey of integrating a custom RoCC accelerator with Chipyard. Please make sure you’re referencing [Chipyard’s docs](https://chipyard.readthedocs.io/en/latest/) if you get stuck.

**Warning:** Be mindful of your Chipyard version. I’m using **v1.10.0**—trust me, it matters. Plus I am using Ubuntu!

---

### The Step-by-Step guide

#### 1. Install Anaconda (Your New Best Friend)

1. Head over to the [Anaconda installer list](https://repo.anaconda.com/archive/) and grab the one that matches your OS.
2. Open your terminal, and let’s run this:
   ```bash
   bash ~/Downloads/Anaconda3-<INSTALLER_VERSION>-Linux-x86_64.sh
   ```
3. Type "yes" as needed (agree to that license, get your Anaconda initialized, etc.). 
4. The default install location? Right here:
   ```bash
   PREFIX=/home/<USER>/anaconda3
   ```

#### 2. Initialize Conda 

```bash
source <PATH_TO_CONDA>/bin/activate
conda init
```

#### 3. Activate Anaconda 

```bash
source ~/.bashrc
```

*(By the way, still using `bashrc`? Not `zshrc`? Are you sure you don’t wanna move on? Just saying...🤣🤣🤣)*

(Optional: Want your shell to auto-activate Anaconda every time it opens? Here you go:)
```bash
conda config --set auto_activate_base True
```

---

#### 4. Install `libmamba` (Because faster solves = less hair-pulling)

```bash
conda install -n base conda-libmamba-solver
conda config --set solver libmamba
conda activate base
```

#### 5. Time to Set Up Chipyard (the real fun begins)

1. Clone the Chipyard repo and check out the right version:
   ```bash
   git clone https://github.com/ucb-bar/chipyard.git
   cd chipyard
   git checkout 1.10.0
   ```
2. Run the setup script and skip the extras (because time is precious):
   ```bash
   ./build-setup.sh riscv-tools -s 6 -s 7 -s 8 -s 9
   ```
   - `-s 6`: Skips FireSim (only if you don’t need it)
   - `-s 7`: Skips FireSim source pre-compilation (trust me, skip it)
   - `-s 8`: Skips FireMarshal (Fire what?)
   - `-s 9`: Skips FireMarshal Linux pre-compilation (we’ve got other things to do)

3. After all that, source the environment script every time:
   ```bash
   source ./env.sh
   ```

   *(Pro tip: Save yourself the trouble by creating an alias in `.bashrc`, still using `bashrc`?)*

   ```bash
   alias chip="cd /home/USER/chipyard/ && source env.sh"
   ```

---

#### 6. Adding Your Custom Chisel Generator (I name it Takhol!)

1. Structure your project directory like this:
   ```
   Takhol/
       build.sbt
       src/main/scala/takhol.scala
   ```

2. Add the project settings to `build.sbt`:
   ```sbt
   organization := "edu.berkeley.cs"
   version := "1.0"
   name := "Takhol"
   scalaVersion := "2.12.13"
   ```

3. Add Takhol to Chipyard’s `generators/` directory:
   ```bash
   cd generators/
   git submodule add https://git-repository.com/Takhol.git
   git submodule update --recursive generators/Takhol
   ```

4. Link Takhol to Chipyard’s top-level `build.sbt` file:
   ```sbt
   lazy val Takhol = (project in file("generators/Takhol"))
     .dependsOn(rocketchip)
     .settings(libraryDependencies ++= rocketLibDeps.value)
     .settings(commonSettings)
   ```

5. Add Takhol as a dependency in the sub-projects section:
   ```sbt
   lazy val chipyard = (project in file("generators/chipyard"))
       .dependsOn(testchipip, rocketchip, boom, hwacha, Takhol, ...)
       .settings(libraryDependencies ++= rocketLibDeps.value)
       .settings(
           libraryDependencies ++= Seq(
           "org.reflections" % "reflections" % "0.10.2"
           )
       )
       .settings(commonSettings)
   ```

---

#### 7. Build with `sbt`

```bash
sbt build
```

#### 8. Add Your Accelerator Config 

Find the `RocketConfigs.scala` file (maybe in `chipyard/generators/chipyard/src/main/scala/config/`) and add your custom config:
```scala
class TakholRocketConfig extends Config (
  new Takhol.WithTakhol ++
  new freechips.rocketchip.subsystem.WithNBigCores(1) ++  // single rocket-core
  new chipyard.config.AbstractConfig
)
```

#### 9. Use Parameters in Takhol.scala 

If the command above seems unclear, and you're unfamiliar with configuring settings in Chipyard or Chisel projects, let me provide a brief overview.

Imagine that in your Takhol.scala file, you define a set of parameters such as rows and columns. To manage these parameters more effectively, you can create a new file named config.scala. This file would look something like this:

```scala
package Takhol

import org.chipsalliance.cde.config._

class WithTakhol extends Config((site, here, up) => {
  case TakholKey => TakholParams(
    rows  = 4,
    columns = 4
  )
})

```

Now, you can use these parameters directly in your Takhol.scala file. Your Takhol.scala might look something like this:

```scala
package Takhol

import chisel3._
import chisel3.util._
import org.chipsalliance.cde.config._

case class TakholParams(
  rows: Int,
  columns: Int)

case object TakholKey extends Field[TakholParams]

abstract trait TakholParameters {
  implicit val p: Parameters

  val takholParams = p(TakholKey)

  val rows = takholParams.rows
  val columns = takholParams.columns
  
}

```

You now have `rows` and `columns` in `Takhol.scala` that you can use them as you wish.

---

#### 10. Set Up RISC-V Simulation with Verilator (Hello, Debugging!)

1. Inside `chipyard/sims/verilator`, create a folder for your program:
   ```bash
   mkdir src
   cd src
   touch hello_world.c
   ```

2. Write your first RISC-V program in `hello_world.c` (simple and classic):
   ```c
   #include <stdio.h>
   
   int main(void) {
       printf("Hello, World!\n");
       return 0;
   }
   ```

3. Build the program: 
   ```
   riscv64-unknown-elf-gcc -fno-common -fno-builtin-printf -specs=htif_nano.specs -c hello_world.c
   riscv64-unknown-elf-gcc -static -specs=htif_nano.specs hello_world.o -o hello_world.riscv
   spike hello_world.riscv
   ```
   If you are lazy like me, create a `run.sh` script to compile and test your code with all baremetal programs:
   
   ```bash
   #!/bin/bash
   
   if [ -z "$1" ]; then
     echo "Usage: sh file.sh <filename.c>"
     exit 1
   fi
   
   filename=$(basename "$1" .c)
   
   echo "riscv64-unknown-elf-gcc -fno-common -fno-builtin-printf -specs=htif_nano.specs -c $filename.c"
   riscv64-unknown-elf-gcc -fno-common -fno-builtin-printf -specs=htif_nano.specs -c "$filename.c" -o "$filename.o"
   
   echo "riscv64-unknown-elf-gcc -static -specs=htif_nano.specs $filename.o -o $filename.riscv"
   riscv64-unknown-elf-gcc -static -specs=htif_nano.specs "$filename.o" -o "$filename.riscv"
   
   echo "spike $filename.riscv"
   spike "$filename.riscv"
   
   
   rm "$filename.o"%     
      
   ```
   and then run it:
   
   ```
   chmod +x run.sh
   ./run.sh hello_world.c 
   ```


   *(If you get an error about `htif_nano.specs`, you missed sourcing `env.sh` earlier. Go back, fix it, and try again!)*

4. Run the simulation with VCD output (yes, you’ll want this):
   ```bash
   make CONFIG=TakholRocketConfig BINARY=src/hello_world.riscv run-binary-debug
   ```

Check the output folder for simulation results, including the `.vcd` file. If you see “Hello, World!” on your terminal, rejoice—everything works. If not... well, you know where to look.

---

### Enjoy the Madness

Congrats! You’re now fully set up to experiment and debug your custom accelerator in Chipyard. Enjoy the chaos and the code!
