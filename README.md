[Install Pandoc](http://pandoc.org/installing.html)

Then:

````bash
$ pandoc InitializeVistA.rst -f rst -t html5 --template vista.html5 -o InitializeVistA.html
$ pandoc InstallCache.rst -f rst -t html5 --template vista.html5 -o InstallCache.html
$ pandoc InstallGTM.rst -f rst -t html5 --template vista.html5 -o InstallGTM.html
$ pandoc InstallVistAOnGTM.rst -f rst -t html5 --template vista.html5 -o InstallVistAOnGTM.html
$ pandoc InstallEWD3.rst -f rst -t html5 --template vista.html5 -o InstallEWD3.html
````