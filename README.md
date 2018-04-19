Plexe data extraction scripts
=

Visit [http://plexe.car2x.org](http://plexe.car2x.org) for information on the
Plexe simulator.

This project includes a set of scripts and configuration files to automatically
extract data from result files obtained through Plexe/Veins/OMNeT++ simulations.

Before trying to run the scripts please make sure you have the following
software installed:

 * `python`
 * `make`
 * `OMNeT++`
 * `R`
 * `reshape2` package for `R`

The folder includes the following files that you are required to use:

 * `README.md`: what you're just reading
 * `genmakefile.py`: a python script that generates the Makefile that will
   automatically perform data extraction for you
 * `parse-config`: this is a config file that you need to modify to tell what you
   want to process and how. More details later
 * `map-config`: this is a config file that dictates the mapping between the
   parameters in the file name of your result files and the name you want to
   assign to those parameters in the post-processed R data file. More details
   later
 * `omnetpp_0.7-1.tar.gz`: this is the omnetpp `R` package that is needed to read
   OMNeT++ result files from within `R`. CAREFUL: this is NOT the version you find
   online, but a version modified by me to re-introduce some features that were
   removed by the original authors. If you have the online version, please make
   sure to remove it and install this version by typing

  `R CMD INSTALL omnetpp_0.7-1.tar.gz`

  on your command line

Other files:

 * `r.tar.bz2`: this is an archive with some OMNeT++ result files that you can use
   to test how the script works. You might want to untar that with

   `tar xf r.tar.bz2`

   to obtain the two `.vec` files `Protocols_2_200_0_0_160_100_9.vec` and
   `Protocols_2_200_1_1_160_1_9.vec`.
 * `generic-parser.R`: utility script
 * `generic-parsing-util.R`: utility script
 * `merge.R`: utility script
 * `omnet_helpers.R`: utility script

WORKING PRINCIPLE
-

The working principle of the script is the following. First, by using
`genmakefile.py`, you generate a `Makefile`, which you can use to extract the data
from your `.vec` files. The idea is that you have one `.vec` file for each
simulation you ran, but from each `.vec` file you might extract different
information you are interested in. For example, if you ran a vehicular
networking simulation, you might have logged mobility data as well as networking
data, and you might want to extract that data into separate files to perform
your post processing and then to draw your plots. By specifying some information
within `parse-config` and `map-config`, you tell the script what you want to extract
and from where. The script then generates a `Makefile` with several targets, one
per each "type" of information you want to extract. Finally, by just typing `make`
in your terminal, all the data you want will be automatically extracted and put
into some `.Rdata` files. Notice that this script DOES NOT perform
post-processing. Post-processing and plotting are two tasks that you will need
to perform on your own depending on what you are working on and what you want to
obtain. This script, however, eases the bothering data extraction procedure.

DETAILED PROCEDURE
-

The required steps are the following:

 1. Edit `map-config` to let the scripts know what you want to extract and how to
    map the parameters in the file name. Inside `map-config` you find several
    sections (marked with []), which define, for each set of information you want
    to extract, how to extract that. In particular, you have the following
    fields:
    - `module`: this defines from which OMNeT++ modules you want to take the data.
      In the sample, you find `scenario.node[*].prot.` If you want to be sure, look
      inside your `.vec` file. For example, look at line 31 within
      `Protocols_2_200_0_0_160_100_9.vec`
    - `names`: this is a comma-separated (no spaces) list of fields you want to
      extract. These are the names you specified with the `setName()` method of your
      `cOutVector`. For example, by specifying `nodeId,busyTime,collisions` you tell
      the script that in your output file you want tuples with the id of the
      node, the channel busy time, and the number of collisions. Notice that for
      this to work, each tuple of values MUST have been logged during a single
      event.
    - `<numbers>`: numbers (like `6 = nCars`) specify the mapping of parameters in
      the `.vec` file name. For example, if you take the following `.vec` filename

      `Protocols_2_200_0_0_160_100_9.vec`

      and the following mapping

      `6 = nCars`

      `8 = runNumber`

      it tells the script that the sixth component (underscore-separated) of the
      filename needs to be called `nCars` (the number of cars in the simulation)
      and the eight needs to be called `runNumber` (the simulation repetition).
      Every unspecified parameter will not be mapped.

    Other mappings in the config can inherit from previously defined mappings.
    For example, in the sample `map-config` file you find

    `[leaderdelay]`

    `inherit = default`

    `names = leaderDelayId,leaderDelay`

    meaning that the mapping should inherit all the parameters of the default
    config, but "`names`" (the data you want to extract) should be different.

 2. Edit `parse-config` to tell the script how to parse the result files. As for
    the `map-config`, the `parse-config` file is divided in sections. First, you have
    a `DEFAULT` section where you can define variables. For example, if you have a
    single `map-config` file for all your configs, you can just define the variable

    `file = map-config`

    and recall it afterwards using `%(file)s`.
    Then you have a `params` section that, for the time being, only includes the
    `resdir` parameter, telling the script where your `.vec` files are located.
    Finally, you have several config sections, one per each kind of information
    you want to extract. The parameters you have in a config are:
    - `config` (e.g., `config = Protocols`): this is the config name of you
      simulation defined in `omnetpp.ini`. This is used to find all the files
      belonging to that config. This means that the file name of every `.vec` file
      you want to process must start with "`Protocols_`".
    - `prefix` (e.g., `prefix = pbc`): each `.vec` file will be parsed according to the
      specified rules and the result will be stored in an `.Rdata` file. For
      example, if the script parses the `Protocols_2_200_0_0_160_100_9.vec` file
      and you specify pbc to be the prefix, the output file will be named
      `pbc.Protocols_2_200_0_0_160_100_9.Rdata`.
    - `out` (e.g., `out = ProtocolsBusyCollisions`): if you decide to merge all your
      results in a single file (see later how), they will be merged in an `.Rdata`
      file with the name specified in your out parameter. In this case, the
      output file will be called `ProtocolsBusyCollisions.Rdata`. This will also be
      the name of the `Makefile` target that will perform the data extraction for
      this config.
    - `map` (e.g., `map = default`): this is the name of the mapping inside the
      `map-config` file that tells which data to extract.
    - `mapFile` (e.g., `mapFile = map-config`): this is the file that includes the
      mapping.
    - `merge` (e.g., `merge = 1` (or `0`)): by setting merge to `0`, the generated
      `Makefile` will NOT merge all the files into a single one. This is useful
      when you have a lot of large output files. In such cases the merge
      operation requires a lot of RAM space and time. If your computer starts
      swapping, it might take forever to conclude the operation. If you have,
      instead, small output files easily manageable, it might be convenient to go
      for the second option. Having a single merged file is very convenient
      because, when doing post-processing and plotting, you have all the data
      that you need at hand. The choice is up to you.
    - `type` (e.g., `type = Rdata` (or `csv`)): by omitting this field or by
      setting it to `Rdata`, the scripts will output binary `Rdata` files that
      can be used with `R`. By setting the value to `csv`, the scripts will
      output comma-separated-values files that can be used with any data
      processing tool.

 3. Generate the `Makefile`. To perform this step run

    `python genmakefile.py parse-config > Makefile`

 4. Run the `Makefile`. If you have a nice terminal with auto completion even for
    make targets, if you type

    `make <TAB><TAB>`

    you will see that, for the sample `parse-config`, there are two targets,
    namely `ProtocolsBusyCollisions.Rdata` and `ProtocolsLeaderDelay.Rdata`. If you
    simply type

    `make`

    and hit enter, the scripts will extract the data for you. When done, you will
    find seven new `.Rdata` files, i.e.,
    - `pbc.Protocols_2_200_0_0_160_100_9.csv`
    - `pbc.Protocols_2_200_1_1_160_1_9.csv`
    - `pld.Protocols_2_200_0_0_160_100_9.Rdata`
    - `pld.Protocols_2_200_1_1_160_1_9.Rdata`
    - `pfd.Protocols_2_200_0_0_160_100_9.Rdata`
    - `pfd.Protocols_2_200_1_1_160_1_9.Rdata`
    - `ProtocolsBusyCollisions.csv`

    The `pbc.*` files are the `.csv` files that extracted `nodeId`, `busyTime`, and
    `collisions` from the two `.vec` files, while the `pld.*` files are the ones that
    extracted `leaderDelayId` and `leaderDelay` from the `.vec` files. Similarly, the
    `pfd.*` files are the ones that extracted `frontDelayId` and `frontDelay`.
    `ProtocolsBusyCollisions.csv` is the merge of the two `pbc.*` files. Given that
    we disabled merging for the `pld.*` and the `pfd.*` files, we get no single
    merged file.

    Notice that the parsing of each `.vec` file is independent, which means that
    you can exploit a multi-core machine and run, for example,

    `make -j 16`

    to run 16 extraction tasks in parallel.
