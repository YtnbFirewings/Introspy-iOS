Introspy
========

Blackbox tool to help understand what an iOS application is doing at runtime
and assist in the identification of potential security issues.


Description
-----------

Introspy comprises two separate modules: a tracer and an analyzer. 

The tracer component can be installed on a jailbroken device and dynamically
configured to hook security-sensitive iOS APIs at run-time. The tool records
details of relevant API calls made by the application, including function
calls, arguments and return values and persists them in a database.
Additionally, the calls can optionally be sent to the Console for real-time
analysis.

The Introspy analyzer can then be used to analyze a database generated by the
tracer, and generate HTML reports containing the list of logged function calls
as well as a list of potential vulnerabilities affecting the application.


Introspy Tracer
---------------

Users should first download the right pre-compiled Debian package:
- https://www.dropbox.com/s/z5cwqk5wti3zsvd/com.isecpartners.introspy-v0.3-iOS_6.1.deb?dl=1

### Dependencies

The tracer will only run on a jailbroken device. Using Cydia, make
sure the following packages are installed:
- dpkg
- MobileSubstrate
- PreferenceLoader
- Applist

### How to install

Download and copy the Debian package to the device; install it:  

    scp <package.deb> root@<device_ip>:~
    ssh root@<device_ip>
    dpkg -i <package.deb>

Respring the device:

    killall -HUP SpringBoard

There should be two new menus in the device's Settings. The Apps menu allows you
to select which applications will be profiled while the Settings menu defines
which API groups are being hooked.

Finally, kill and restart the App you want to monitor.

### How to uninstall

    dpkg -r com.isecpartners.introspy

Introspy Analyzer
-----------------

The analyzer requires Python 2.6 or 2.7.

### Usage

The Introspy tracer should be first used on the application to be tested, i.e.,
by selecting it within the "Introspy - Apps" Settings menu. Then simply specify
the device IP address when you run the analysis tool and select the appropriate
application database. This will store a local copy of the database, which you
can analyze again by specifying the database name as opposed to the device IP
address.

    $ python introspy.py 192.168.1.127 --outdir e-bank
    mobile@192.168.1.127's password:
    0. ./Applications/94656731-0259-4AE9-9EEE-BADC9244AD82/introspy-com.isecpartners.e-bank.db
    1. ./introspy-com.apple.mobilemail.db
    2. ./introspy-com.isecpartners.introspytestapp.db
    Select the database to analyze: 0

The example above will generate an HTML report for the com.isecpartners.e-bank
application within the newly created "e-bank" directory (specified by the
`--outdir` option). The HTML report is intended to be the most common interface to
the call database and allows users to browse the full call list or filter the
list to view only those calls flagged by specific signatures.

#### Signatures

Beyond simply listing the calls recorded by the Introspy tracer, the analysis
tool allows you to apply predefined signatures to the call list and flag
potential vulnerabilities or insecure configurations. Users can browse the list
of flagged calls simply by browsing to the "Potential Findings" view within the
generated HTML report and expanding the desired signature group.

The signatures themselves are defined in `analyzer/Signatures.py` and can be
easily extended. The following example adds a signature to identify NSData file
writes that don't include data protection values. Beyond simply identifying
method calls, argument matching and argument existence filters can also be
applied.

    signature_list.append(Signature(
    title = 'Lack of File Data Protection With NSData',
    description = 'A file was written without any data protection options.',
    severity = Signature.SEVERITY_MEDIUM,
    filter = MethodsFilter(
        classes_to_match = ['NSData'],
        methods_to_match = ['writeToFile:atomically:', 'writeToURL:atomically:'])))

### Command-line Usage

#### Reporting

While the HTML formatted report is the most digestable format, the analysis tool
can also be used directly from the command-line. Just as the HTML report allows
you to show/hide signature groups and subgroups, you can specify groups (-g) as
well as subgroups (-s) when running the analysis to limit the output to only
those calls that match the filtering criteria.

    $ python introspy.py introspy-com.isecpartners.e-bank.db -g IPC -s Schemes
    Specific URL schemes are implemented by the application.
        CFBundleURLTypes:CFBundleURLSchemes
		arguments =>
		  CFBundleURLIsPrivate => nil
		  CFBundleURLName => transfer-money
		  CFBundleURLScheme => transfer-money

This example shows analysis of a local database with filtering options to limit
the output to only display registered URL schemes. We can see here that URL
requests with the transfer-money:// scheme will be handled by the application.

The analysis tool also allows users to print the entire call list similarly to
the HTML report's "Traced Calls" view by specifiying the `--list` option,
although this will print an undigestable amount of data to stdout and as such is
not recommended.

#### Enumerations

The command-line tool also allows users to enumerate various data from the list
of traced calls (via `--info`), inlcuding a list of all of the unique URLs
accessed by the application (http), all files accessed (fileio), as well as
Keychain items that were added or modified (keys).

    $ python introspy.py introspy-com.isecpartners.e-bank.db --info keys
	token = MGJiNzg1NGRkNzBkNGMyZTExNzc4NTA3OTdjNjNkNjFiY2Q1
	consumerKey = YzAwNzE4ZDZlYjYzOTM4NGM2NTc56j
	consumerSecret = NmUzYmNjNmQ2YjJjNWU1MDE0Zjk3NGI4MzU4ZWRl

### Programmatic Usage

    >>> from argparse import Namespace
    >>> import introspy
    >>> spy = introspy.Introspy(Namespace(db='introspy-com.isecpartners.e-bank.db', group='IPC', sub_group='Schemes', list=None))
    >>> for call in spy.analyzer.tracedCalls:
    ...   print call.json_encode()
    ...
    {"class": "CFBundleURLTypes", 
     "method": "CFBundleURLSchemes"}, 
         "arguments": 
            {"CFBundleURLName": "transfer-money", 
             "CFBundleURLScheme": "transfer-money", 
             "CFBundleURLIsPrivate": "nil"}
    }

Doing It Yourself
-----------------

### Building the Tracer From Source

Most users should just download and install the pre-compiled Debian package.
However, if you want to modify the library's functionality you will have to
clone the source repository and build the debian package yourself.

    git clone https://github.com/iSECPartners/introspy.git

The build requires the Theos suite to be installed; 
see http://www.iphonedevwiki.net/index.php/Theos/Getting\_Started .
You first have to create a symlink to your theos installation:

    cd introspy/tracer/
    ln -s /opt/theos/ ./theos

Then, the package can be built using:

    make package

### Installing the Tracer From Source

Once you've successfully created the debian package, you can use the Theos
Makefiles to automatically install the package and respring the device:

    export THEOS_DEVICE_IP=192.168.1.127
    make install

Group and Subgroup Filtering
----------------------------

The groups and subgroups correlate to filtering via the Settings menu as well as
during offline analysis using the command-line. For details on exactly which
methods correspond to each group and subgroup, refer to the wiki
[documentation](https://github.com/iSECPartners/introspy/wiki).

License
-------

See ./LICENSE.

Authors
-------

* Tom Daniels
* Alban Diquet
