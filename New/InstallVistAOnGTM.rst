Download VistA for GT.M
=======================

Authors: Sam Habiel

License: 

.. image:: https://i.creativecommons.org/l/by/4.0/80x15.png 
   :target: http://creativecommons.org/licenses/by/4.0/ 

A Mumps database (like VistA) is a series of routines and globals (a global
in Mumps really means a file on disk). To load VistA into GT.M, you need to
obtain the these from the CACHE.DAT distributed by the VA. Efforts are
underway to lobby the VA to distribute the FOIA instance as a set of globals
and routines; rather than in a proprietary format.

Since the establishment of OSEHRA, each update monthly update of FOIA is
exported as routines and globals in zwrite format at `Github <https://github.com/OSEHRA/VistA-M>`_.
In addition, DSS vxVistA can be obtained from `this repository <https://github.com/OSEHRA/vxVistA-M>`_
and WorldVistA can be obtained from `this repository <https://github.com/glilly/wvehr2-dewdrop>`_.

In our example, for setting up a VistA Database, we will use FOIA VistA.

Before downloading VistA, we will start by creating an empty database.

Creating an Empty GT.M Database suitable for VistA
--------------------------------------------------
Create a directory where you will place your environment. These two steps need
to be done as a superuser. The directory name or location doesn't matter. In this case,
it's ``foia201608`` under ``/var/db``. Second step changes the ownership to your
user name and your user group.

.. raw:: html
    
    <div class="code"><code>$ <strong>sudo mkdir -p /var/db/foia201608</strong>
    $ <strong>sudo chown $USER:$USER /var/db/foia201608</strong>.
    $ <strong>cd /var/db/foia201608</strong></code></div>

Then create folders to hold your routines, globals, journals, and objects. The
convention in the VistA community is to call these folders r g j and o. While it's
easier to say routines globals journals and objects, I do not want to break with
convention.

.. raw:: html
    
    <div class="code"><code>$ <strong>mkdir r g j o</strong></code></div>

Two parenthetical remarks:

    FIS (the company behind GT.M) recommends versioning objects
    and global directories to allow for rolling upgrades. I personally don't 
    think this is necessary for VistA. More details can be found at the
    `GT.M Acculturation Workshop <https://sourceforge.net/projects/fis-gtm/files/GT.M%20Acculturation%20Workshop/>`_.
    
    Various people in the VistA community create another directory
    called "p" for patches, so that you can apply updated  routines
    in the "r" directory and not overwrite the original routine. The intent is
    reasonable, but what what almost always happens is that I get calls or emails
    on why changes aren't showing up. VistA tools (KIDS, Fileman, VPE) are all
    written just expecting a single routine list.

At this point, we need to create an environment file that we will need to
source in order to tell GT.M where are our routines and globals are. The reason
we need to do this is simple: GT.M bases its operations almost entirely on
environment variables from the shell. Here's the file, which I called `env.vista<./env.vista>`_.

.. raw:: html
    
    <div class="code"><code> # This is just a variable so I don't have to type the same thing
    # over and over again.
    export vista_home="/var/db/{your directory name}"
    
    # This will set the prompt. This can be anything you want.
    # I make it something meaningful to let me know which environment I am on.
    export gtm_prompt="YOUR INSTANCE NAME>"
    
    # Intial Value of error trap upon VistA start-up
    #export gtm_etrap='W !,"ERROR IN STARTUP",!! D ^%ZTER HALT' # for production environments
    export gtm_etrap='B'             # for development environments
    
    # The location of the global directory. A global directory tells GT.M in
    # which database file we will locate a global
    export gtmgbldir="${vista_home}/g/mumps.gld"
    
    # The location of where GT.M was installed. 
    # You may need to adjust this based on where you installed it
    export gtm_dist="/usr/lib/fis-gtm/current/"     
    
    # Where the routines are. 
    # If you run 32 bit GT.M, you need to remove libgtmutil.so
    # On older versions of GT.M (&lt;6.2), the * isn't recognized.
    # There should be no reason for you to run 32-bit GT.M these days.
    export gtmroutines="${vista_home}/o*(${vista_home}/r) $gtm_dist/libgtmutil.so"
    
    # Allow relink of routine even if it is on the stack
    export gtm_link="RECURSIVE"
    
    # Adjust QUIT behavior to accommodate Cache bug/feature of 
    # C style function/procedure unification rather than M/Pascal style 
    # function/procedure dichotomy
    export gtm_zquit_anyway=1
    
    # Run this routine when a process is asked to interrogate itself
    # using mupip intrpt
    export gtm_zinterrupt='I $$JOBEXAM^ZU($ZPOS)'
    
    # GT.M has non-standard default behavior for null subscripts for local
    # variables. Make it standard
    export gtm_lct_stdnull=1
    
    # Add GT.M to the path if not already there.
    [[ ":$PATH:" != *":${gtm_dist}"* ]] && export PATH="${PATH}:${gtm_dist}"
    
    # GT.M should not short-cut $SELECT and binary boolean operators
    # A default optimization.
    export gtm_side_effects=1
    export gtm_boolean=1
    
    # $SYSTEM Output to use to identify the box the system is running on
    export gtm_sysid="foia.2016.08"</code></div>
    
Once this is done, source the file using ``$ . env.vista``. Then test that
what you did works by running ``$ mumps -dir``. You should see this:

.. raw:: html
    
    <div class="code"><code>YOUR INSTANCE NAME></code></div>

Type Control-D or "HALT" to get out.

Now we need to create the database. You can create a default database by just
running ``mupip create``, but rather than do that, we need to write some code
to tell GT.M to change its default database for VistA. I will create a file 
called `g/db.gde<./db.gde>`_.

.. raw:: html
    
    <div class="code"><code>! Change the default segment's file 
    ! to be g/mumps.dat
    ! to have 4096 byte blocks
    ! to have an initial DB size of 1048576*4096=4GB
    ! to allow 1000 locks
    ! On production environments, add -extension_count=0 to prevent the database
    ! -> from growing automatically. You need to monitor it and expand it yourself.
    ! -> Here, it extends by 100MB each time.
    ! Global buffer count is how many buffers of size block_size should stay in
    ! -> RAM to cache the data read and written to disk. This set-up uses about 33MB in RAM.
    change -segment DEFAULT -file="$vista_home/g/mumps.dat" -access_method=BG -allocation=1048576  -block_size=4096 -lock_space=1000 -global_buffer_count=8192 -extension_count=25600
    
    ! Ditto pretty much, except this is smaller. Note that we create a new segment
    ! rather than modify an existing one.
    ! TEMPGBL unlike the others will be memory mapped to the RAM to allow instant
    ! access.
    ! Since it's located in RAM, global_buffer_count does not apply to it.
    add    -segment TEMPGBL -file="$vista_home/g/tempgbl.dat" -access_method=MM -allocation=10000   -block_size=4096 -lock_space=1000 -extension_count=2560
    
    ! Each global node can be 1024 bytes long; subscripts can be combined to be 512 bytes long
    ! You will need to increase this for RPMS
    change -region  DEFAULT -record_size=1024 -stdnullcoll -key_size=512
    
    ! Ditto, but note that we need to assign the new region to its associated segment
    add    -region  TEMPGBL -record_size=1024 -stdnullcoll -key_size=512 -dynamic=TEMPGBL
    
    ! Add globals to the temporary region
    add    -name    HLTMP   -region=TEMPGBL
    add    -name    TMP     -region=TEMPGBL
    add    -name    UTILITY -region=TEMPGBL
    add    -name    XTMP    -region=TEMPGBL
    add    -name    BMXTMP  -region=TEMPGBL
    add    -name    XUTL    -region=TEMPGBL
    add    -name    VPRHTTP -region=TEMPGBL
    add    -name    ZZ*     -region=TEMPGBL
    
    ! show all for verification
    show -all

    ! save
    exit</code></div>

Once you save the file, run it.

.. raw:: html
    
    <div class="code"><code>$ <strong>mumps -run ^GDE < g/db.gde |& tee g/db.gde.out</strong></code></div>

A successful invocation will show you this output on the screen and saved into
g/db.gde.out as well.

.. raw:: html
        
    <div class="code"><code>

    %GDE-I-GDUSEDEFS, Using defaults for Global Directory 
      /var/db/foia0616/g/mumps.gld

    GDE> 

                                   *** TEMPLATES ***
                                                                              Std      Inst
                                                 Def     Rec   Key Null       Null     Freeze   Qdb      Epoch
     Region                                     Coll    Size  Size Subs       Coll Jnl on Error Rndwn    Taper
     -----------------------------------------------------------------------------------------------------------
     <default>                                     0     256    64 NEVER      N    N   DISABLED DISABLED ENABLED

     Segment          Active              Acc Typ Block      Alloc Exten Options
     ------------------------------------------------------------------------------
     <default>          *                 BG  DYN  1024        100   100 GLOB =1024
                                                                         LOCK = 40
                                                                         RES  =   0
                                                                         ENCR = OFF
                                                                         MSLT =1024
                                                                         DALL=YES
     <default>                            MM  DYN  1024        100   100 DEFER
                                                                         LOCK = 40
                                                                         MSLT =1024
                                                                         DALL=YES

             *** NAMES ***
     Global                             Region
     ------------------------------------------------------------------------------
     *                                  DEFAULT
     BMXTMP                             TEMPGBL
     HLTMP                              TEMPGBL
     TMP                                TEMPGBL
     UTILITY                            TEMPGBL
     VPRHTTP                            TEMPGBL
     XTMP                               TEMPGBL
     XUTL                               TEMPGBL
     ZZ*                                TEMPGBL

                                    *** REGIONS ***
                                                                                                    Std      Inst
                                     Dynamic                          Def      Rec   Key Null       Null     Freeze   Qdb      Epoch
     Region                          Segment                         Coll     Size  Size Subs       Coll Jnl on Error Rndwn    Taper
     ----------------------------------------------------------------------------------------------------------------------------------
     DEFAULT                         DEFAULT                            0     1024   512 NEVER      Y    N   DISABLED DISABLED ENABLED
     TEMPGBL                         TEMPGBL                            0     1024   512 NEVER      Y    N   DISABLED DISABLED ENABLED

                                    *** SEGMENTS ***
     Segment                         File (def ext: .dat)Acc Typ Block      Alloc Exten Options
     -------------------------------------------------------------------------------------------
     DEFAULT                         $vista_home/g/mumps.dat
                                                         BG  DYN  4096    1048576 25600 GLOB=8192
                                                                                        LOCK=1000
                                                                                        RES =   0
                                                                                        ENCR=OFF
                                                                                        MSLT=1024
                                                                                        DALL=YES
     TEMPGBL                         $vista_home/g/tempgbl.dat
                                                         MM  DYN  4096      10000  2560 DEFER
                                                                                        LOCK=1000
                                                                                        RES =   0
                                                                                        ENCR=OFF
                                                                                        MSLT=1024
                                                                                        DALL=YES

                                      *** MAP ***
       -  -  -  -  -  -  -  -  -  - Names -  -  - -  -  -  -  -  -  -
     From                            Up to                            Region / Segment / File(def ext: .dat)
     --------------------------------------------------------------------------------------------------------------------------
     %                               BMXTMP                           REG = DEFAULT
                                                                      SEG = DEFAULT
                                                                      FILE = $vista_home/g/mumps.dat
     BMXTMP                          BMXTMP0                          REG = TEMPGBL
                                                                      SEG = TEMPGBL
                                                                      FILE = $vista_home/g/tempgbl.dat
     BMXTMP0                         HLTMP                            REG = DEFAULT
                                                                      SEG = DEFAULT
                                                                      FILE = $vista_home/g/mumps.dat
     HLTMP                           HLTMP0                           REG = TEMPGBL
                                                                      SEG = TEMPGBL
                                                                      FILE = $vista_home/g/tempgbl.dat
     HLTMP0                          TMP                              REG = DEFAULT
                                                                      SEG = DEFAULT
                                                                      FILE = $vista_home/g/mumps.dat
     TMP                             TMP0                             REG = TEMPGBL
                                                                      SEG = TEMPGBL
                                                                      FILE = $vista_home/g/tempgbl.dat
     TMP0                            UTILITY                          REG = DEFAULT
                                                                      SEG = DEFAULT
                                                                      FILE = $vista_home/g/mumps.dat
     UTILITY                         UTILITY0                         REG = TEMPGBL
                                                                      SEG = TEMPGBL
                                                                      FILE = $vista_home/g/tempgbl.dat
     UTILITY0                        VPRHTTP                          REG = DEFAULT
                                                                      SEG = DEFAULT
                                                                      FILE = $vista_home/g/mumps.dat
     VPRHTTP                         VPRHTTP0                         REG = TEMPGBL
                                                                      SEG = TEMPGBL
                                                                      FILE = $vista_home/g/tempgbl.dat
     VPRHTTP0                        XTMP                             REG = DEFAULT
                                                                      SEG = DEFAULT
                                                                      FILE = $vista_home/g/mumps.dat
     XTMP                            XTMP0                            REG = TEMPGBL
                                                                      SEG = TEMPGBL
                                                                      FILE = $vista_home/g/tempgbl.dat
     XTMP0                           XUTL                             REG = DEFAULT
                                                                      SEG = DEFAULT
                                                                      FILE = $vista_home/g/mumps.dat
     XUTL                            XUTL0                            REG = TEMPGBL
                                                                      SEG = TEMPGBL
                                                                      FILE = $vista_home/g/tempgbl.dat
     XUTL0                           ZZ                               REG = DEFAULT
                                                                      SEG = DEFAULT
                                                                      FILE = $vista_home/g/mumps.dat
     ZZ                              Za                               REG = TEMPGBL
                                                                      SEG = TEMPGBL
                                                                      FILE = $vista_home/g/tempgbl.dat
     Za                              ...                              REG = DEFAULT
                                                                      SEG = DEFAULT
                                                                      FILE = $vista_home/g/mumps.dat
     LOCAL LOCKS                                                      REG = DEFAULT
                                                                      SEG = DEFAULT
                                                                      FILE = $vista_home/g/mumps.dat
    GDE> 
    GDE> 
    GDE> 
    %GDE-I-VERIFY, Verification OK

    %GDE-I-GDCREATE, Creating Global Directory file 
    /var/db/foia0616/g/mumps.gld
    </code></div>

If you fail, you will see something similar to the following at the end of the
output:

.. raw:: html
    
    <div class="code"><code>%GDE-I-VERIFY, Verification FAILED
    
    %GDE-E-VERIFY, Verification FAILED</code></div>

At this point, we are ready to create our databases. This is easy.

.. raw:: html
    
    <div class="code"><code>$ <strong>mupip create</strong>
    Created file /var/db/foia201608/g/mumps.dat
    Created file /var/db/foia201608/g/tempgbl.dat</code></div>

To check that everything works fine, run ``mumps -dir`` and then ``DO ^%GD``
and ``DO ^%RD``. The first will open all the database files for searching and
open a shared memory segment on your machine. The second will make sure that
your ``$gtmroutines`` variable is correct.

.. raw:: html
    
    <div class="code"><code>$ <strong>mumps -dir</strong>
    
    FOIA 2016-08><strong>D ^%GD</strong>
    
    Global Directory
    
    Global ^<strong>*</strong>
    
    Total of 0 globals.
    
    Global ^<strong>&lt;enter&gt;</strong>
    
    FOIA 2016-08><strong>D ^%RD</strong>
    
    Routine directory
    Routine: <strong>*</strong>
    
    Total of 0 routines.
    
    Routine: <strong>&lt;enter&gt;</strong></code></div>

It's common with all Unix software relying on POSIX/SysV Shared Memory to
report errors with ``shmget()``. If you see that when you are trying to run ^%GD, 
you need to increase your shared memory limits. I will leave you to google
that on your own.

Loading VistA Into the GT.M Database we just Created
----------------------------------------------------
I said we will use FOIA VistA. Make sure that wget is installed on your
machine, and then get the code (takes 3-30 minutes depending on your internet
connection). First switch to a working directory (such as /tmp/) and run this:

.. raw:: html
   
    <div class="code"><code>$ <strong>wget https://github.com/OSEHRA/VistA-M/archive/foia.zip</strong></code></div>

If you want WorldVistA or vxVistA instead, you can get them from
https://github.com/glilly/wvehr2-dewdrop/archive/master.zip or
https://github.com/OSEHRA/vxVistA-M/archive/master.zip respectively.

Now unzip it:

.. raw:: html
   
    <div class="code"><code>$ <strong>unzip foia.zip</strong></code></div>

Everything gets unzipped in the folder `VistA-M-foia/`, so you need to use that
folder as the first argument of the find commands below.

Next we need to copy the routines to VistA (takes about 30 seconds). There are
quotes around the ``{}`` because the paths contain spaces.

.. raw:: html
    
    <div class="code"><code>$ <strong>find VistA-M-foia/ -name '*.m' -exec cp "{}" r/ \;</strong></code></div>

Next we need to load the globals. We use the versatile ``mupip load`` command
for that. Note that mupip load wants quotes sent down from the shell for any
paths that contain spaces; and these do. Again, we tee our output because there
is so much of it and because we need to visually inspect that everything got
loaded. This takes time; from 10 minutes up to 30 minutes.

.. raw:: html
    
    <div class="code"><code>$ <strong>find VistA-M-foia -name '*.zwr' -exec echo {} \; -exec mupip load \"{}\" \; |& tee g/foia201608-load.log</strong></code></div>

Verify that none of the globals failed to import.

.. raw:: html
    
    <div class="code"><code>$ <strong>fgrep '%GTM' g/foia201608-load.log | wc -l</strong></code></div>

If you get an output that isn't zero, you need to visually inspect what
happened.

After we are done with this, we will repeat our smoke test with %GD and %RD.

.. raw:: html
    
    <div class="code"><code>$ <strong>mumps -dir</strong>
    
    FOIA 2016-08><strong>D ^%GD</strong>
    
    Global Directory
    
    Global ^<strong>*</strong>
    
    ...
    
    Total of 391 globals.
    
    FOIA 2016-08><strong>D ^%RD</strong>
    
    Routine directory
    Routine: <strong>*</strong>
    ...
    Total of 35547 routines.</code></div>

At this point we are done loading VistA. It's time to enable journaling on
all the regions we want. That can be a separate script, but I put it with my
env script so that everything can be in one place and I only have to source
one file to activate my VistA instance. On a production VistA instance, you
would create an initd or systemd script to do this rather than put it into the
environment script.

Here's the environment script example first. This is good for demo/dev databases.
Add this to the end. This recovers the database if it was journaled and then
enables journaling. File here: `vista.journaling<./vista.journaling>`_

.. raw:: html
    
    <div class="code"><code># This is journaling.
    if [ -f ${vista_home}/j/mumps.mjl ]; then
      if (( $(lsof -t ${vista_home}/g/mumps.dat | wc -l) == 0 )); then
        $gtm_dist/mupip journal -recover -backward ${vista_home}/j/mumps.mjl
      fi
    fi

    if (( $(find ${vista_home}/j -name '*_*' -mtime +3 -print | wc -l) > 0 )); then
        echo "Deleting old journals"
        find ${vista_home}/j -name '*_*' -mtime +3 -print -delete
    fi

    if (( $(lsof -t ${vista_home}/g/mumps.dat | wc -l) == 0 )); then
      $gtm_dist/mupip set -journal="enable,on,before,f=${vista_home}/j/mumps.mjl" -region DEFAULT
    fi</code></div>

Source the env.vista script again to enable journaling.

If you would rather create an init script, here's an example to copy. This
provides much more functionality than journaling--it's the kind of thing you
would have on a production instance.

NB: You need to put a valid value for vista_instance and the user also needs
to be valid (here vistauser). File here: `vista.initd<./vista.initd>`_

.. raw:: html

  <div class="code"><code>#!/usr/bin/env bash
  #---------------------------------------------------------------------------
  # Copyright 2011-2012 The Open Source Electronic Health Record Agent
  #
  # Licensed under the Apache License, Version 2.0 (the "License");
  # you may not use this file except in compliance with the License.
  # You may obtain a copy of the License at
  #
  #     http://www.apache.org/licenses/LICENSE-2.0
  #
  # Unless required by applicable law or agreed to in writing, software
  # distributed under the License is distributed on an "AS IS" BASIS,
  # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  # See the License for the specific language governing permissions and
  # limitations under the License.
  #---------------------------------------------------------------------------

  # init script for VistA
  # Modifications by SMH for VistA

  # Debian LSB info
  ### BEGIN INIT INFO
  # Provides:          foiavista
  # Required-Start:    $remote_fs $syslog
  # Required-Stop:     $remote_fs $syslog
  # Default-Start:     2 3 4 5
  # Default-Stop:      0 1 6
  # Short-Description: Start VistA services at boot time
  # Description:       Starts/Stops VistA instances in a sane way.
  ### END INIT INFO


  # Start VistA
  vista_instance="/path/to/your/vista"
  start() {
    source ${vista_instance}/env.vista
    su - vistauser -c "source ${vista_instance}/env.vista &&
      if [ -f ${vista_home}/j/mumps.mjl ]; then
        echo \"Recovering old journals...\"
        mupip journal -recover -backward ${vista_home}/j/mumps.mjl
      fi"

    if (( $(find ${vista_home}/j -name '*_*' -mtime +3 -print | wc -l) > 0 )); then
      echo "Deleting old journals..."
      find ${vista_home}/j -name '*_*' -mtime +3 -print -delete
    fi

    su - vistauser -c "source ${vista_instance}/env.vista; mupip rundown -region '*'" 
    su - vistauser -c "source ${vista_instance}/env.vista; mupip set -journal=\"enable,on,before,f=${vista_home}/j/mumps.mjl\" -region DEFAULT"
    su - vistauser -c "source ${vista_instance}/env.vista; mumps -run %XCMD 'J ZISTCP^XWBTCPM1(9210)'" 
    su - vistauser -c "source ${vista_instance}/env.vista; mumps -run ZTMB"
    su - vistauser -c "source ${vista_instance}/env.vista; mumps -run %XCMD 'J START^XOBVLL(8000)'"
  }

  # Stop VistA
  stop() {
      su - vistauser -c "source ${vista_instance}/env.vista; mumps -run %XCMD 'S U=\"^\" D STOP^ZTMKU' << EOF
  Y
  Y
  Y
  EOF"
      # Wait for TaskMan to stop
      echo "Waiting for TaskMan to stop (30 sec)"
      sleep 30

      echo "Stopping any remaining M processes nicely"
      su - vistauser -c ". ${vista_instance}/env.vista && pgrep mumps | xargs --max-args=1 mupip stop"
      sleep 2
      
      processes=$(pgrep mumps)
      if [ ! -z "${processes}" ] ; then
        echo "M process are being shutdown forcefully!"
        pkill -9 mumps
      fi
      rm -fv /tmp/gtm_*
  }

  case "$1" in
      start)
          start
          ;;
      stop)
          stop
          ;;
      restart)
          stop
          start
          ;;
      *)
          echo "Usage: $0 {start|stop|restart}"
          ;;
  esac</code></div>

You have to save this script in /etc/init.d/, and make it execuatble and owned
by root, and add it the correct run levels for the Linux kernel. On Ubuntu,
this would look like this. You need to be root (or sudo) to perform these
steps:

.. raw:: html

    <div class="code"><code>$ <strong>cd /etc/init.d/</strong>
    $ <strong>edit vista.initd</strong> # create the file here. Skip if done.
    $ <strong>chown root:root vista.initd</strong>
    $ <strong>chmod +x vista.initd</strong>
    $ <strong>update-rc.d vista.initd defaults</strong>
    $ <strong>update-rc.d vista.initd enable</strong></code></div>

The next step is not necessary if you don't plan to have users log-in. You should
pre-compile the routines on GT.M so they do not have to be compiled at runtime.

.. raw:: html

    <div class="code"><code>$ cd o
    $ for r in ../r/*.m; do mumps $r; done 2>&1 | tee ../compile_all.log
    </code></div>

At this point, you are ready to continue to `Initialize Vista
<./InitializeVistA.html>`_.
