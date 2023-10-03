# VTXCart
Reverse engineering Neo Geo 161 in 1 cartridge to change Rom games.

## Introduction
This fork repo was created mainly to document and help to track my process of re-programming one of thise 161 in 1 MVS cartridges.

Before anything, I would like to thank for the contributions and research made by **xvortex** and **rockbottom** before him. Without their work, this project would be impossible.

The inspiration came from [xvortex's post](https://www.arcade-projects.com/threads/reverse-engineering-161-in-1-cartridge-to-change-rom-games.15069/page-13#post-395646) at an arcade-projects thread that aimed to flash new games in place of the 50 or so bootlegs/clones originally shipped with those Multicards sold at Chinese marketplaces like Ali and Wish. 

The main problem is that due to the nature of the reverse engineering done, there's a lot of guff and technical mumbo-jumbo that's way to complex for the average hobbyist to make sense of. My hope is that at the end of my journey, I will have laid out a nice tutorial on how to do everything from requirements all the way to the final assembly.

## Disclaimer
As of the last few commits (dated 23/10/01 and later), this documentation is in Super-Mega-Alpha-Stage, DO NOT FOLLOW ALONG if you managed to find this repo. At the current stage, I'm waiting for my PCBs, flashers and female connectors to arrive, so there will be a few weeks (if that) for me to get to a stage where I can write the juicy soldering steps.

**This README is basically just a glorified notepad of my observations.**

Before I start to lay down the tutorial, it is important to set some expectations of what is currently possible (and impossible) to do. If you don't want the technical stuff and want to jump right into the action, you can skip this and go straight to [here](#getting-started)

When we talk about "replacing the games" in these carts what we actually will be doing is reprogramming the various ROM chips that store different types of game assets with new information. Original NEO GEO game cartridges are split into several "partitions" that stores different kinds of "stuff" like the sprites, audio, game logic etc. This goes beyond the scope of this writing so if you want to learn more about the anatomy of a game I suggest you to read [this](https://wiki.neogeodev.org/index.php?title=Cartridges) article. Suffice it to say that each game is split into different memory locations, and the multicard NEEDS to abide by these limits if it wants to be read by the console.

The current crop of these multicards have a total of 8 ROM chips that have the memory divided to these sizes:

C-ROM (2 chip 1 GB each) = 2GB.<br>
P-ROM (3 chip 128 MB each) = 384 MB.<br>
V-ROM (1 chip) = 1 GB.<br>
M-ROM (256 "slots" by 256 KB each).<br>
S-ROM (256 slots by 128 KB each).<br>

While it was discovered that the default images flashed into some of these chips leaves A LOT of free space, we cannot use some of the allocated memory for one type of ROM into another (for example, using only 512MB of V-ROM and allocating the other half to P-ROM).

The main concern in this case is the C-ROM. All Metal Slugs + KOF's + Samurai Shodown's take approximately 982 MB of C-ROM data. If we extrapolate to the total MVS library that makes about 2.7 GB in terms of C-ROM, which means that **it is impossible to fit the entire romset into a single cartridge.** The current line of thinking is to make two cartridges, one with those bigger titles and another with everything else.

Lastly, while the current method will allow you to flash basically the majority of the known library, due to the limited number of macrocells in the CPLD we cannot (at least now) implement some specific bankswitches on them (NEO-CMC, NEO-PVC, etc.) as well as decryptors. For some games there are patches that disable these things, for some you will have to take bootlegs. Alas, the entire original ROMset simply will not fit into this CPLD architecture without changes.

With all that out of the way, let's start!

## Requirements
  - Advanced soldering skills (writing this for future readers, but yea fair to say this is not for beginners)
  - Compatible [161 in 1 V3 Multicard](https://aliexpress.com/item/1005004288835061.html?spm=a2g0o.order_list.order_list_main.29.36c2caa4Sedyn3&gatewayAdapt=glo2bra) (obviously) 
  - [WeAct Studio based on the STM32H750](https://aliexpress.com/item/1005001475058305.html?spm=a2g0o.cart.0.0.59807f06RCRzWF&mp=1&gatewayAdapt=glo2bra) (the one with the little LCD display)
  - [USB Blaster](https://aliexpress.com/item/1005002912036038.html?spm=a2g0o.cart.0.0.59807f06RCRzWF&mp=1&gatewayAdapt=glo2bra)  (CPLD Programmer)
  - 6x [2.54mm dual-row female headers](https://aliexpress.com/item/1005003980057698.html?spm=a2g0o.cart.0.0.59807f06RCRzWF&mp=1&gatewayAdapt=glo2bra) (2x22 pins)**\***
  - 8x [1.27mm dual-row female headers](https://aliexpress.com/item/1005003815207542.html?spm=a2g0o.cart.0.0.59807f06RCRzWF&mp=1&gatewayAdapt=glo2bra) (2x22 pins)**\***
  - 9x 0.1uF SMD 0603 ceramic caps  
  - At least 1 of each custom made PCBs inside **./Dumpers/Gerbers**

  **\*** These sellers didn't have the 2x22pin at the time I ordered. You can buy all of these at your local reseller of choice but if you go along with these you can just cut them to size.

  **PAY ATTENTION TO THE CORRECT F0095H0 BOARD FOR YOUR SPECIFIC MULTICARD!** Some versions have an custom stilt that solders to the main PBC by through-hole DIP connectors, others just solder the chips right on. Inside the Gerbers folder you will find a .zip for either option but you only need the appropriated for your multicard, not both. For my tutorial, I will use the DIP one.

## Getting started - Generating the ROM files 
First things first, clone this repo into an easy-to-reach folder on your computer with the ```git clone``` command. After that, go into the **./Compiler/Games** folder. Inside you will find a bunch of folders with the majority of the MVS/AES library with the MAME convention name scheme; within each folder you can find a few different files:

* **bram** is just a blank file for games that store saving data into the backup ram present in all NEO GEO hardware.
* **fpga** is the file that contain the specific code that will instruct the Altera Max on how to interpret the final ROM data and sent down the info to the console itself. You can find a more detailed explanation of how to write it at **./Compiler/Doc/fpga.txt**.
* **menu** is how the game will be displayed at the final menu.

Aside from them, the folders don't contain any of the required ROM data for the compiler to work. You will need to supply your own ROMs formatted into the *Darksoft format*. To make it easier, there's a bunch of tools to convert vanilla dumps into the Darksoft format, [here's an example](https://github.com/aluzed/MiSTer-Neogeo-Rom-Decrypter). This is the same format used by the MiSTer FPGA. If any particular game / homebrew / bootleg isn't present in this folder you may need to create your own folder and fill it with new files; doing so might need some extra research into what parameters each file will need.

Due to limitations of the hardware, the ROMs will need to de decompressed and decrypted. There are a bunch of collections and romsets available on the internet that already have most of the work done. This may or may not be on the final tutorial, but lets just say that a friend of mine gave you [this](https://archive.org/download/multipacks) link right here and told you to download the 2.2GB one 😉

After placing all the necessary files into each folder, the final result needs to look something like this:

[a screenshot of a completed folder]

Next, go into the **./Compiler/bin** folder. Inside you will find two .txt files:

* **games_all.txt** is the list of all the games the compiler will be aware at the timing of making the files. If you added new folders at the previous step, to the same inside this file.
* **games.txt** is the list of the games that will actually be compiled to the final files. THE ORDER MATTERS! whatever order you place your games here will be the same in the final menu.

For the simplicity of this step of the tutorial, I will generate only these games:

[a screenshot of the used games.txt]

After adding all your desired games, type ```cmd``` at the navbar of the explorer window to open a prompt at the current folder. After that, type ```_run.bat``` to start the compile process:

[a screenshot of the cmd running _run.bat]

When the VTXCart.exe start doing it's job, it will do a pre-validation step to make sure the final output will be error-free. If it finds anything out of place it will crash out and display an message of what need to be done. There can be a few possibilities:

* A folder referenced in **./Compiler/bin/games.txt** couldn't be found at **./Compiler/Games**. Either remove it from the list or add the folder.
* A file inside one of **./Compiler/Games** folders wasn't found. Either remove the game from the list or add the missing files.
* A particular game uses an unsupported bank switching mode. Either find an patched or bootlegged version of the same game online or remove it from the list.
* The maximum size of the available ROM chips was excedded in X MB. Remove some games from the list until it fits.
*TODO: Are there more erros? Which?*

The process itself might take a few minutes depending on the quantity of the games and the speed of your processor. If everything went fine it end the script and display an ```Done.``` message: 

[a screenshot of the cmd saying done]

At the end of the process, there should be three new folders: **MAME**, **ROM** and **Verilog**:

[a screenshot of the cmd saying done]

Inside **./Compiler/bin/ROM**

## Assembling the adapter PCBs
The first thing to do before anything is to get the PCBs manufactured. I will not go into detail on how you do this, but a basic Google search will get you sorted.

*TODO: Wait for everything to arrive and build the PCBs, desolder the chips and instruct on how to flash each one of them*

## Cliff notes (as of my current understanding)
Everything below is just my conclusions as I'm barreling along with this project (23/10/01), take everything with a grain of salt.

The original repo is subdivided within the following folders:
  - **Compiler**: xvortex's tool that takes the files in each folder and *compiles* to the formats required to be flashed at each chip by the end. More details at it's specific section.
  - **CPLD**: (Complex Programmable Logic Device) There are three Altera Max FPGAs (aka. CPLDs) inside each cartridge, two on the PROG board and one at the CHAR board. They are responsible for reading the data on each ROM chip and translate to a format that the NEO GEO recognizes. The files at this folder seems to be the raw, unedited files that the VTXCart.exe uses to compile the final images to be flashed to each chip. **DO NOT ALTER OR CHANGE ANYTHING HERE!!**
  - **Docs**: Schematics and datasheets of the chips and boards found at a variety of Multicards. These are most likely a byproduct of reverse engineering and were committed for archival purposes. Nothing of interest for the overall process.
  - **Dumpers**: General files for getting the chips connected to the respective programmers and actually reading/writing data onto them
    -- *TODO*: figure out (aside from the Gerbers folder) how the content of each folder works.
  - **Excel**: Also a byproduct of the reverse engineering, both files contain more schematics and were used for mapping each connection from the ROM chips to the rest of PCB. This was important at the time since bootleggers LOVE to scramble the vias in order to make figuring out how they did it that much more difficult than it needed to be. Overall this can be ignored as well.
  - **Photo**: Photos of the boards :)
