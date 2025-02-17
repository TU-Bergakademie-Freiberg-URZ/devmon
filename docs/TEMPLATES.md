
 DEVMON TEMPLATES
 ==============================================================================


 What are templates?
 ------------------------------------------------------------------------------

 Templates are one of the key concepts behind Devmon, which make it uniquely
 flexible when compared to many other monitoring scripts.

 Templates allow you to configure the way Devmon treats your devices, on a per-
 model basis. They allow you define the following specific items for different
 model types:

 - Preferred SNMP version(s)
 - The specific OIDs to query on (both repeater and non-repeaters)
 - Any transformations needed to be preformed on collected data
 - Model specific thresholds
 - Model specific exceptions
 - Custom output messages

 This flexibility should (in theory) allow a savvy systems administrator to
 monitor any type of device for any possible condition; the only pre-requisite
 is that the device be SNMP monitorable in the first place!



 Rolling your own
 -------------------------------------------------------------------------------

 Creating a Devmon template is (I hope) a relatively trivial task. It requires
 no programming experience; however you will most likely benefit from knowing
 a little about regular expressions.

 If you aren't familiar with regular expressions (or 'regexps') you should
 take a few minutes and look over this website: http://www.regular-
 expressions.info/

 Okay, first we examine the template file structure. All templates data is
 located in the "templates" subdirectory of your Devmon installation. A
 single node installation of Devmon reads this directory once per poll-cycle,
 while a multi-node installation reads it from the Devmon database on an as-
 needed basis.

 For all our examples below, we will assume that we are working in the
 template directory "/usr/local/devmon/templates". This will probably differ
 in your installation, depending on where you have installed Devmon.

 The first tier of subdirectories in the templates dir are vendor-model
 specific. So, each subdirectory represents a particular model from a
 particular vendor (so, a Cisco 2950 would have one directory, while a Cisco
 3750 would have another). Any files in the templates directory will be
 ignored; only directories are examined. The actual name of these directories
 is irrelevant, as the actual vendor and model names are specified in the
 'specs' file (described below). However, it doesn't hurt to make the
 subdirectories somewhat descriptive; I usually use a "vendor-model" style.

 Note:

 In Multinode, I would recommend that you only keep a single copy of your
 templates directory, preferably the one on your display server. All others
 templates directories (i.e. the ones on your Devmon nodes) are extraneous and
 should be removed. This way, when you sync your templates on disk to your
 database, there's no confusion as to which set of templates on disk match the
 ones that a multinode Devmon installation is using (which are the ones in the
 database).


 The 'specs' file
 ------------------------------------------------------------------------------

 Under each vendor-model directory, there should be a single
 file, named 'specs' (for specifications), and one or more
 subdirectories, each of which represent a particular test for that
 vendor-model.  The specs file contains data that is vendor-model 
 specific, but not necessarily test specific.

 The 'specs' file should look something like this:

  <!--start file--------------------------->
  ```
  vendor   : cisco
  model    : 2950
  snmpver  : 2
  sysdesc  : C2950
  ```
  <!----end file--------------------------->

 Note that their variables and their values are each listed on their own
 newline, and separated by colons. This is the format used for most (if not
 all) of the files in the Devmon template structure.

 The 'vendor' and 'model' variables are specific to this particular device
 type -- there should not be another specs file elsewhere in the templates
 tree that has the same values for both variables. If there is, Devmon will
 complain about trying to redefine a template and reject the second template.

 The 'snmpver' variable is DEPRECATED and can be safely removed from all
 templates as it is not used anymore.

 The 'sysdesc' variable is used by the type auto-detection that Devmon does
 when it initially reads the host from the bb-hosts file (when using the --
 readbbhosts command line argument). This value should be unique when compared
 to the value of the other templates. It's a regular expression, so you can
 match a fairly complicated pattern, if you so desire.


 Test directory
 ------------------------------------------------------------------------------

 Each subdirectory of the vendor-model directory represents an individual
 test. The name of the directory is significant, as it is what determines the
 name of the test reported to your display server! So the subdirectory in your
 vendor-model directory named 'cpu' defines the cpu test, the one named
 'if_err' defines the if_err test, etc.

 Under each test subdirectory, there are five files:

    oids
    transforms
    thresholds
    exceptions
    message

 All five files MUST be present for the template to be read successfully,
 although the thresholds, transforms and exceptions files can all be empty.
 So, a quick list of files needed for a 'cpu' test on a Cisco 2950 should look
 as follows:

    /usr/local/devmon/templates/cisco-2950/specs
    /usr/local/devmon/templates/cisco-2950/cpu/oids
    /usr/local/devmon/templates/cisco-2950/cpu/transforms
    /usr/local/devmon/templates/cisco-2950/cpu/thresholds
    /usr/local/devmon/templates/cisco-2950/cpu/exceptions
    /usr/local/devmon/templates/cisco-2950/cpu/message

 Note that all of these files except for the message file can contain
 comments. Any line that starts with a pound symbol (#) is treated as a
 comment by Devmon, and ignored.

 Now we'll go over each of these files in detail...


 The 'oids' file 
 ------------------------------------------------------------------------------

 The oids file contains, you guessed it, the oids that you want to SNMP query
 for this type of device. It should look something like this:

 <!--start file--------------------------->

    sysDescr        : .1.3.6.1.2.1.1.1.0               : leaf
    sysReloadReason : .1.3.6.1.4.1.9.2.1.2.0           : leaf
    sysUpTime       : .1.3.6.1.2.1.1.3.0               : leaf
    CPUTotal5Min    : .1.3.6.1.4.1.9.9.109.1.1.1.1.5.1 : leaf

 <!----end file--------------------------->

 Note that there are three values per line; the first value is the alias that
 Devmon uses throughout the rest of the template files, the second value is
 the *NUMERIC* value for the oid, the third is the repeater type ('leaf',
 which is a non-repeater type oid, vs 'branch' which is a repeater type).

 Its important that you use the numeric version of an oid for the second value
 in this file. Devmon will not map the string version of an OID to its numeric
 version before it does a query, which means that your SNMP query will fail if
 you use an alphanumeric oid instead of a numeric one (i.e. 'sysDescr' is
 alphanumeric, '.1.3.6.1.2.1.1.1.0' is numeric). I chose to do this because it
 is a pain to keep all the various MIBs installed on all of the nodes in a multi-
 node cluster, and it was just easier to specify them once here. Note that the
 oid aliases are case sensitive: 'SysDescr' is treated as a separate alias
 from 'sysDescr'.

 Also important to note is that OIDs are shared between tests on the same
 template. So if you specify OID aliases with identical names (they are case
 sensitive, remember) in multiple tests in a template, there is only going to
 be a single value stored in memory, which both OID aliases point to. The
 upshot of this is, if you use the same OID alias in multiple tests (and this
 is recommended, as it will make your template run faster), then they *MUST*
 have the numeric OID value. If they dont, you are going to get inconsistent
 results, as the value stored in memory might arbitrarily be from one SNMP
 variable or another.


 ### The 'OID' concept

 More explaination of the 'OID' term as it is a key concept that should be
 well understood.

 SNMP standard and Devmon:

 - In SNMP, we have 'table' OIDs and 'scalar' OIDs
 - In Devmon, we have OIDs of type 'branch' and 'leaf'

 The relation between the both is:

 - A 'branch' OID is a snmp 'table'
 - A 'leaf' OID can be:
   - A snmp 'scalar' OID (end with 0)
   - An instance of a snmp 'table' OID (do not end with 0, normally...)


  How it works:
  <!-- A query from the shell -->
  ```
  snmpwalk -v2c -c public MYDEVICE .1.3.4.6.9
  .1.3.4.6.9.4.3.1.20.3 = 8732588786
  .1.3.4.6.9.4.3.1.20.4 = 5858738454
  ```

  Let's analyse the answer:
  ```
  .1.3.4.6.9.4.3.1.20.3 = 8732588786
  <-  oid -> <- index-> = <-result->
  ```

 There are 2 information on each line:

 - The 'result' can be of different types: String, Integer, OID, ...
 - The 'index' has type: OID (most of the time it is an Integer)

 So depending on the type of our OID we have:

    Branch: result[idx] = snmp(oid)    ->  n indexes (n>=0) -> n results
    Leaf  : result      = snmp(oid)    ->  1 result (but no index)

 The meaning of the OID can be confusiong as it is the variable to be polled
 AND the polling 'result'.

    oid(idx) = snmp(oid)   
    oid      = snmp(oid)

 Note: If a leaf OID is an instance (element) of an OID. Try to transform it
       to a 'branch'!

 - it's more scalable
 - it's probably be more efficient (TODO: add an example like: hp- ilo/cpu_dm).

 The OID term is also use in Devmon to designate the result of a transform:

    oidT=transform{oidS} 

 Note that in Devmon a transform can have multiple oids:
 - oidT = transform {oidS1, oidS2, ... }
 - The resulting OID have the same indexes as the 'primary oid'
   - Indexes of oidT are the same as the source oids: oidS1
   - The primary oid has to be choosen as the result depend on it! 

 Conclusion:
 - 'source oid(s)' is/are what is polled or transformed
   - primary oid, is the 'first' source oids 
 - 'target oid' is the result after a transformation or a polling
   - 'target oid indexes' are the the indexes of a table returned by snmp 
   - 'target oid values' are the results returned by snmp


 The 'transforms' file
 ------------------------------------------------------------------------------

 The most complicated file in your template, the transforms file lays out the
 different data transformations that Devmon needs to perform on the collected
 SNMP data before it applies thresholds and renders the final message.

 The cisco 2950 cpu test uses a very simple transforms file:

    <!--start file--------------------------->

    sysUpTimeSecs   : MATH          : {sysUpTime} / 100
    UpTimeTxt       : ELAPSED       : {sysUpTimeSecs}

    <!----end file--------------------------->

 Like the oids file, it has three values per line, separated by colons.

 The first value is the OID alias. Note that this should be a unique value
 compared to any of the aliases defined in the 'oids' file. Notice in this
 example that the 'sysUpTimeSecs' alias is a transformed version of the
 'sysUpTime' alias, which was defined in the oids file and whose data is
 collected via SNMP. For the rest of this help file, we use the term 'alias'
 to interchangeably refer to either a variable containing data collected via
 SNMP or containing data from a translation.

 The second value in the line is the name of the type of transform. These are
 case insensitive (i.e. 'MATH' is the same as 'math') but we refer to them in
 the uppercase form to distinguish them from other functions.

 The third value defines the 'data input' for the transform specified by the
 second value. The result of this data put through the specified transform
 will be stored in the alias defined by the first value. Note that any aliases
 supplied in this field are encased in curly braces (e.g. {sysUpTime}). This
 tells Devmon that this is an alias containing an snmp or translated value,
 and not just a normal string.

 The data input for a transform can (depending on the transform type) consist
 of one or more OID aliases defined elsewhere. These aliases don't have to
 necessarily be defined in a line prior the transform that they are used in,
 Devmon is smart enough to figure out the hierarchy in which they should be
 used. If you have a dependency loop somewhere, Devmon will point that out to
 you, as well.

 Also, note that if you use a non-repeater type data alias as the input for
 a transform, the transformed alias will also be a non-repeater. Likewise
 for a repeater type data alias. If you mix repeater and non-repeater type
 data aliases in the transform input, the resulting transformed alias will
 be a repeater.

 With regard to duplicated OID aliases across multiple tests in a single
 template, transformed OID aliases have the same rules as non-transformed
 aliases: if you use the same transformed OID alias in multiple tests (which
 is recommended as this cuts down on the time Devmon spends running test
 logic) then their transform rules *must be identical*, as must all OID
 aliases that your transformed alias depends on. So, for example, if you have
 this defined in your if_load test on your cisco-2950 template:

    ifInBps         : MATH          : {ifInOps} x 8

 and this defined in your if_stat test on your cisco-2950 template:

    ifInBps         : MATH          : {ifInOctets} x {time} x 8

 you are going to be in trouble, because the 'time' OID alias might not even
 exist in the if_load test. So try to keep your duplicated OID aliases as
 simple as possible, so you dont have your tests stepping on each others toes
 (although if you do have two transformed OIDs doing the same transform on the
 same data, you should by all means duplicate them, as this will make your
 tests run much faster).

 There are a number of different types of transforms, which we will discuss
 below: (listed in alphabetical order)


 ### BEST transform
 This transform takes two data aliases as input, and stores
 the values for the one with the 'best' alarm color (green being the 'best'
 and red being the 'worst') in the transformed data alias. The oids can either
 be comma or space delimited.


 ### CHAIN transform

 Occasionally a device will store a numeric SNMP oid (AKA the 'data' oid) as a
 string value under another OID (the 'leaf' oid). The CHAIN transform will
 create a third 'transformed' oid, containing the leaves of the 'leaf' oid and
 the values of the 'data' oid. A quick example:

 In your oids file, you have defined:

    leafOid  : .1.1.2     : branch
    dataOid  : .1.1.3     : branch

 After walking leafOid and dataOid, they return the values:

    .1.1.2.1 = '.1.1.3.1194'
    .1.1.2.2 = '.1.1.3.2342'

 and

    .1.1.3.1194 = 'CPU is above nominal temperature'
    .1.1.3.2342 = 'System fans are non-operational'

 Chances are that you won't know what leaf values will be returned for
 .1.1.3, but you know that .1.1.2 returns consistent values. You can use the
 CHAIN transform to 'chain' these two oids together to make the data more
 accessible. The format for the CHAIN transform is:

    chainedOid   : CHAIN    : {leafOd} {dataOid}

 If you used the above transform with the previously mentioned data, you
 would end up with:

    chainedOid.1 = 'CPU is above nominal temperature'
    chainedOid.2 = 'System fans are non-operational'


 ### CONVERT transform

 Convert a string in either hexadecimal or octal to its decimal equivalent.
 Takes two arguments, a target OID alias and a conversion type, which must be
 either 'hex' or 'oct'.

 For instance, to convert the hex string '07d6' to its decimal equivalent
 (2006, as it so happens), do this:

    intYear : CONVERT: {hexYear} hex


 ### DELTA transform

 The DELTA transform performs a 'change over time' calculation on the
 supplied data. It takes a single data alias, with an optional 'upper limit'
 (separated from the alias by whitespace) as input.

 The change over time calculation will be performed between one poll interval
 and the next, and returns a measurement of data units per second.

 The limit is used as the maximum value of the data alias, and comes in to
 play when the value from supplied data alias from the last polling cycle is
 more than the value from your current polling cycle.

 This typically occurs when you have counter-wrap issues in SNMP (as most
 counters are still 32 bit; an interface with heavy traffic can wrap its
 ifOctet counters in less than two minutes).

 If you don't specify a limit and Devmon detects a counter- wrap, it will use
 either the 32bit or 64bit upper limit, accordingly.

 The upshot of this is that you CANNOT MEASURE NEGATIVE DELTAS WITH THIS
 TRANSFORM. If you really really need to, please contact the software author
 and make a feature request.

 Delta examples:

    changeInValue  : DELTA : {value}

 or

    changeInValue  : DELTA : {value} 2543456983

 Keep in mind that the DELTA transform takes at least two poll cycles to
 return meaningful data, so in the mean time you will get a 'wait' result
 stored in the target OID alias (as well as in aliases that are transformed
 based off the target alias).


 ### DATE transform

 This transform takes a single data alias as input, the value of which Devmon
 assumes to be seconds in "Unix time" (i.e. seconds since the Epoch [00:00:00
 GMT, January 1, 1970]) It then stores in the transformed data alias a text
 string containing the date corresponding to the number of seconds input, in
 the format CCYY-MM-DD, HH:MM:SS (24 hour time).


 ### ELAPSED transform

 This transform takes a single data alias as input, the value of which Devmon
 assumes to be in seconds. It then stores a text string in the transformed
 data alias containing the number of years, days, hours, minutes and seconds
 equal to the number of seconds provided as input to the transform.


 ### INDEX transform

 This transform allows you to access the index part of a numerical OID in a
 repeater OID.

 For example, in the cdpCache table for the Cisco CDP MIB, walking the
 cdpCacheDevicePort OID will return values such as:

    CISCO-CDP-MIB::cdpCacheDevicePort.4.3 = STRING: GigabitEthernet4/41
    CISCO-CDP-MIB::cdpCacheDevicePort.9.1 = STRING: GigabitEthernet2/16
    CISCO-CDP-MIB::cdpCacheDevicePort.12.14 = STRING: Serial2/2

 The value is the interface on the remote side, and there is no OID for the
 interface on the local side. To get the interface on the local side, you must
 use the last value in the index (e.g. 3 for GigabitEthernet4/41) and look in
 the ifTable:

    IF-MIB::ifName.3 = STRING: Fa0/0

 The index transform allows you to get the index value (4.3 in this case) as
 an OID value. Any operations you need to do on the index value should be
 possible with existing transforms.

 ### MATCH transform

 In some badly designed MIBs multiple types of information are presented in a
 single table with two columns (branches), often in just a name, value format.
 This transform makes it possible to split such a combined table out into
 separate tables, or to reformat the table so that it has multiple columns.

 For example, the MIB for the TRIDIUM building management system has a table
 with outputName and outputValue, data returned looks as follows:

    TRIDIUM-MIB::outputName.1  = STRING: "I_Inc4_Freq"
    TRIDIUM-MIB::outputName.2  = STRING: "I_Inc4_VaN"
    TRIDIUM-MIB::outputName.3  = STRING: "I_Inc4_VbN"
    TRIDIUM-MIB::outputName.4  = STRING: "I_Inc4_VcN"
    ...
    TRIDIUM-MIB::outputValue.1 = STRING: "50.06"
    TRIDIUM-MIB::outputValue.2 = STRING: "232.91"
    TRIDIUM-MIB::outputValue.3 = STRING: "233.39"
    TRIDIUM-MIB::outputValue.4 = STRING: "233.98"

 To split the frequences out as a separate repeater, use:

    outputFreqRow  : MATCH  : {outputName} /.*_Freq$/
    outputVaRow    : MATCH  : {outputName} /.*_VaN$/
    ...

 outputFreqRow will now contain the indexes of outputName that matched the
 regular expression, e.g. 1,5,9 etc. , outputVaRow will contain 2,6,10. To
 construct a table, use the chain transform to create repeaters using the
 matched indexes:

    outputFreq     : CHAIN  : {outputFreqRow} {outputValue}
    outputVa       : CHAIN  : {outputVaRow} {outputValue}
    ...

 To create the primary repeater for a table, we do the same on outputName:

    IncomerRowName : CHAIN  : {outputFreqRow} {outputName}   

 In this case, it is preferable to clean up the outputFreq for display:

    IncomerName    : REGSUB : {IncomerRowName} /(.*)_Freq/$1/

 A table created as follows:

    Incomer|Frequency (Hz)|Voltage A|Voltage B|Voltage C

 Would now contain in its first row:

    I_Inc4|50.06|232.91|233.39|233.98   

 'MATH' transform:

 The MATH transform performs a mathematical expression defined by the
 supplied data. It can use the following mathematical operators:

    '+'           (Addition)
    '-'           (Subtraction)
    '*'           (Muliplication)
    ' x '         (Multiplication - note white space on each side) (deprecated)
    '/'           (Division)
    '^'           (Exponentiation)
    '%'           (Modulo or Remainder)
    '&'           (bitwise AND)
    '|'           (bitwise OR)
    ' . '         (string concatenation - note white space each side)
    '(' and ')'   (Expression nesting)

 This transform is not whitespace sensitive, except in the case of ' x ' and
 ' . ' , so both:

    {sysUpTime} / 100

 and

    {sysUpTime}/100

 ...would be accepted, and are functionally equivalent. However:

    {ifInOps} x 8

 will work, while:

    {ifInOps}x8

 will not. This is to avoid problems with oid names containing the character
 'x'. New templates should rather use the '*' operator to avoid problems, e.g.:

    {ifInOps}*8

 The mathematical expressions you can perform can be
 quite complex, such as:

    ((({sysUpTime}/100) ^ 2 ) x 15) + 10

 Note that the syntax of the MATH transform is not stringently checked at
 the time the template is loaded, so if there are any logic errors, they
 will not be apparent until you attempt to use the template for the first
 time (any errors will be dumped to the devmon.log file on the node that
 they occurred on).

 Decimal precision can also be controlled via an additional variable seperated
 from the main expression via a colon:

     transTime : MATH : ((({sysUpTime}/100) ^ 2 ) x 15) + 10 : 4 

 This would ensure that the transTime alias would have a precision value (zero
 padded, if needed) of exactly 4 characters (i.e. 300549.3420). The default
 value is 2 precision characters. To remove the decimal characters
 alltogether, specify a value of 0.

 ### UNPACK transform 
  
 The inverse of the 'PACK' transform.

 ### REGSUB transform

 One of the most powerful and complicated transforms, the regsub transform
 allows you to perform a regular expression substitution against a single data
 alias input. The data input for a regsub transform should consist of a single
 data alias, followed by a regular expression substitution (the leading 's'
 for the expression should be left off). For example:

    ifAliasBox : REGSUB  : {ifAlias} /(\S+.*)/ [$1]/

 The transform above takes the input from the ifAlias data alias and, assuming
 that it is not an empty string (ifAlias has to have at least one non-
 whitespace character in it) it puts square braces around the value and puts a
 space in front of it. This example is used by all of the Cisco interface
 templates included with Devmon, to include the ifAlias information for an
 interface, but only if it has a value defined. A very powerful, but easily
 misused transform. If you are interested in using it but don't know much
 about substitution, you might want to google 'regular expression
 substitution' and try reading up on it.

 ### SET transform

 The SET transform creates a repeater-type OID, and presets it with a sequence
 of constants. The indexes of the individual values of the OID created by SET
 are numbered starting from 1. The constants are defined in the third field.
 The constants are separated by a comma, optionally surrounded by zero or more
 spaces. Leading and trailing spaces in the list of constants are ignored. At
 least one constant must be specified. A constant is either a number or a
 string of characters, which should not include ',', '{' or '}'. It is not
 possible to define spaces at the start or at the end of the string.

 Like the MATCH transform, the SET transform is meant to be used for a badly
 designed MIB. While MATCH is used if two branches are used, containing the
 name and the value, SET is used if only one branch is used, containing a list
 of values.

 For example, the MIB for the McAfee MEB 4500 contains a section describing (a
 part of) the file systems. For each file system, the utilisation of space,
 the size, the free space, the utilisation of the i-nodes, the total number of
 i-nodes and the number of free i-nodes are available. A better representation
 is a table with 6 columns. The following configuration is used to map the
 single column onto 6 columns.

     fsInfo    : .1.3.6.1.4.1.1230.2.4.1.2.3.1 : branch

     fsiUtil   : SET    : 11.0,17.0,23.0,29.0,35.0,41.0
     fsiSize   : SET    : 12.0,18.0,24.0,30.0,36.0,42.0
     fsiFree   : SET    : 13.0,19.0,25.0,31.0,37.0,43.0
     fsiIUtil  : SET    : 14.0,20.0,26.0,32.0,38.0,44.0
     fsiISize  : SET    : 15.0,21.0,27.0,33.0,39.0,45.0
     fsiIFree  : SET    : 16.0,22.0,28.0,34.0,40.0,46.0

     fsbName   : SET    : deferred,quaratine,scandir,logs,var,working
     fsbUtil   : CHAIN  : {fsiUtil} {fsInfo}
     fsbSize   : CHAIN  : {fsiSize} {fsInfo}
     fsbFree   : CHAIN  : {fsiFree} {fsInfo}
     fsbIUtil  : CHAIN  : {fsiIUtil} {fsInfo}
     fsbISize  : CHAIN  : {fsiISize} {fsInfo}
     fsbIFree  : CHAIN  : {fsiIFree} {fsInfo}

 The OIDs named fsb.+ can be used in transforms and in the TABLE directive in
 file 'message'.

 From a theoretical stand point, the SET transform complements the set of
 transforms. There was already the possibility to set a leaf-type OID to a
 constant value, using a statement like:

    AScalar  : MATH   : 123

 The SET transform introduces the same possibility for a repeater-type OID.


 ### SPEED transform

 This transform takes a single data alias as input, which it assumes to be a
 speed in bits. It then stores a value in the transformed data alias,
 corresponding to the largest whole speed measurement. So a value of 1200
 would render the string '1.2 Kbps', a value of 13000000 will return a value
 of '13 Mbps', etc.

 ### STATISTIC transform

 This transform takes a repeater type data alias as the input for the
 transform and computes a non-repeater type data alias. The STATISTIC
 transform can compute the minimum value, the maximum value, the average value
 and the sum of the values of the repeater type data alias. Moreover it can
 count the number of values of the repeater type data alias.

 If the input is a non-repeater data alias, the transform returns the value of
 the input data. However, if the number of values is to be counted the
 returned value is 1.

 If for example the average temperature in a device with multiple temperature
 sensors is to be monitored, the transformation could be:

    TempAvg : STATISTIC : {ciscoEnvMonTemperatureStatusValue} AVG

 As the example shows, the last keyword determines the value to be returned.
 The possible keywords are:

    AVG : Average value
    CNT : Number of values
    MAX : Maximum value
    MIN : Minimum value
    SUM : Sum of the values

 ### SUBSTR transform

 The substr transform is used to extract a portion of the text (aka a
 'substring') stored in the target OID alias. This transform takes as
 arguments: a target alias, a starting position (zero based, i.e. the first
 position is 0, not 1), and an optional length value. If a length value is
 not specified, substr will copy up to the end of the target string.

 So, if you had an OID alias 'systemName' that contained the value 'Cisco
 master switch', you could do the following:

    switchName : SUBSTR : {systemName} 0 12

 stores 'Cisco master' in the 'switchName' alias, or

    switchName : SUBSTR : {systemName} 6

 stores 'master switch' in the 'switchName' alias


 ### SWITCH transform

 The switch transform transposes one data value for another. This is most
 commonly used to transform a numeric value returned by an snmp query into its
 textual equivalent. The first argument in the transform input should be the
 oid to be transformed. Following this should be a list of comma- delimited
 pairs of values, with each pair of values being separated by an equals sign.

 For example: 

    upsBattRep : SWITCH : {battRepNum} 1 = Battery OK, 2 = Replace battery

 So this transform would take the input from the 'upsBattRepNum'
 data alias and compare it to its list of switch values. If
 the value of upsBattRepNum was 1, it would store a 'Battery OK'
 value in the 'upsBattRep' data alias. 

 You can use simple mathematical tests on the values of the source OID
 alias, as well as assigning values for different OIDs to the target alias.
 For instance:

    dhcpStatus : SWITCH : {dhcpPoolSize} 0 = No DHCP, >0 = DHCP available

 The format for the tests are as follows (assuming 'n','a' and 'b' are
 floating point numerical value [i.e. 1, 5.33, 0.001, etc], and 's' is a
 alphanumeric string):

    n       : Source alias is equal to this amount
    >n      : Source alias is greater than this amount
    >=n     : Source alias is greater than or equal to this amount
    <n      : Source alias is less than this amount
    <=n     : Source alias is less than or equal to this amount
    a - b   : Source alias is between 'a' and 'b', inclusive
    's'     : Source alias matches this string exactly (case sensitive)
    "s"     : Source alias matches this regular expression (non-anchored)
    default : Default value for the target alias, used in case none of
              the other statements match (what about ,=?)

 Note that switch statements are applied in a left to right order; so if you
 have a value that matches the source value on multiple switch statements, the
 leftmost statement will be the one applied.

 The switch statement can also assign values from another OID to the target
 OID alias, depending on the value of the source OID alias, like this:

    dhcpStatus : SWITCH : {dhcpPoolSize} 0 = No DHCP, >0 = {dhcpAvail}

 This would assign the value 'No DHCP' to the 'dhcpStatus' alias if and only
 if the 'dhcpPoolSize' alias contained a value equal to zero. Otherwise, the
 value of the 'dhcpAvail' alias would be assigned to dhcpStatus. Note that
 threshold stats for the 'dhcpAvail' status (i.e. the 'color' and 'message'
 assigned to 'dhcpAvail' by any threshold tests) would not be inherited by the
 'dhcpStatus' variable; if you want to inherit threshold information, use the
 TSWITCH transform instead.


 ### TSWITCH transform (DEPRECATED)

 The TSWITCH transform is functionally equivalent to the SWITCH transform in
 every way, with one exception: if any OID alias is used as a data source for
 the target alias, such as the 'dhcpAvail' alias in this transform:

    dhcpStatus : TSWITCH : {dhcpPoolSize} 0 = No DHCP, >0 = {dhcpAvail}

 The threshold values for that alias will be copied to the target alias (in
 this case, 'dhcpStatus'), and no further thresholds will be applied to the
 target alias. Any non-OID data sources can still have thresholds applied
 against them (for instance, if 'dhcpStatus' had been assigned the string 'No
 DHCP' by this transform, you could have matched a threshold against that
 value). This is useful if you have two seperate OIDs and you need to do a
 compound threshold involving them both.

 ### UNPACK transform

 The unpack transform is used to unpack binary data into any one of a number
 of different data types (all of which are eventually stored as a string by
 Devmon). This transform requires a target OID alias and an unpack type (case
 sensitive), separated by a space.

 As an example, to unpack a hex string (high nybble first), try this:

    hexString : UNPACK : {binaryHex} H

 The unpack types are as follows:

    Type  |  Description              
    ---------------------------------------------------
       a  | ascii string, null padded
       A  | ascii string, space padded
       b  | bit string, low to high order
       B  | bit string, high to low order
       c  | signed char value
       C  | unsigned char value
       d  | double precision float
       D  | single precision float
       h  | hex string, low nybble first
       H  | hex string, high nybble first
       i  | signed integer
       i  | unsigned integer
       l  | signed long value
       L  | unsigned long value
       n  | short integer in big-endian order
       N  | long integer in big-endian order
       s  | signed short integer
       S  | unsigned short integer
       v  | short integer in little-endian order
       V  | long integer in little-endian order
       u  | uuencoded string
       x  | null byte
       

 ### WORST transform

 This transform takes two data aliases as input, and stores the values for the
 one with the 'worst' alarm color (red being the 'worst' and green being the
 'best') in the transformed data alias. The oids can either be comma or space
 delimited.


 The 'thresholds' file
 ------------------------------------------------------------------------------

 The thresholds file defines the limits against which the various data aliases
 that you have created in your 'oids' and 'transforms' files are measured
 against. An example thresholds file is as follows:

 -<start file>-----------------------------

    upsLoadOut  : red     : 90          : UPS load is very high
    upsLoadOut  : yellow  : 70          : UPS load is high

    upsBattStat : red     : Battery low : Battery time remaining is low
    upsBattStat : yellow  : Unknown     : Battery status is unknown

    upsOutStat  : red     : On battery|Off|Bypass              : {upsOutStat}
    upsOutStat  : yellow  : Unknown|voltage|Sleeping|Rebooting : {upsOutStat}

    upsBattRep  : red     : replacing : {upsBattRep}

 -<end file>-------------------------------

 As you can see, the thresholds file consists of one entry per line, with each
 entry consisting of three to four fields separated by colons. The first field
 in an entry is the data alias that the threshold is to be applied against.
 The second field is the color that will be assigned to the data alias should
 it match this threshold. The third field has the threshold values, which are
 the values that the data alias in the first field will be compared against.
 You can have multiple values, delimited by vertical bars, in the third field.
 The fourth field is the threshold message, which will be assigned to the data
 alias in the first field if it matches this threshold.

 The threshold message can contain other data alias(es) (oids): if they are of a 
 branch type they have to share indexes with the first field. If they are leafs
 they do not need as there is only one value. The threshold field cannot use
 data aliases (oids) value (this is feature request). 

 Starting with version 0.20.1
 - We can have multiple error message for a color
 - The threshold engine change it behaviour
  
 New threshold engine v0.20.1
 0. Split the threshold field with the comma delimiter 
 1. Evaluate the precision of the threshold based on its operator
    7. =, eq
    6. > >= < >=
    5. ~= (smart match)
    4. !~  (negative smart match)
    3. !=, ne
    2. _AUTOMATCH_
    1. (empty)
 2. Then the order from highest severity to the lowest red->yellow->clear->green 
 
 This change normnally is mainly compatible with previous implementation, but
 but add greater flexibility

 One thing to note about thresholds is that they are lumped into one of twoi
 categories: numeric and non-numeric
 - Some operateur like Smart match only appy to non-numeric. 
 - Numeric operator are evaluated first

 If no math operator is defined in the threshold, Devmon assumes that it is a
 'greater than' type threshold. That is, if the value obtained via SNMP is
 greater than this threshold value, the threshold is considered to be met
 and Devmon will deal with it accordingly. This is ambiguous with '='. This 
 should be avoid and replace with the '>' operator. (Should raised a warning 
 TODO: make a deprecation notice)

 If a threshold value contains even one non-numeric character (other than the
 math operators illustrated above), it is considered a non-numeric threshold.

 Regular expressions in threshold matches are non-anchored, which means they
 can match any substring of the compared data. So be careful how you define
 your thresholds, as you could match more than you intend to! If you want to
 make sure your pattern matches explicitly, precede it with a '^' and
 terminate it with a '$'.


 The 'exceptions' file
 ------------------------------------------------------------------------------

 The exceptions file is contains rules which are only applied against repeater
 type data aliases.

 An example of a exceptions file is as follows:

 -<start file>-----------------------------

    ifName : alarm  : Gi.+
    ifName : ignore : Nu.+|Vl.+

 -<end file>-------------------------------

 You can see that each entry is on its on line, with three fields separated by
 colons. The first field is the primary data alias that the exception should
 be applied against. The second field is the exception type, and the third
 field is the regular expression that the primary alias is matched against.
 Exception regular expressions (unlike non-numeric thresholds) ARE anchored,
 and thus need to match the primary oid EXACTLY.

 Exceptions are only applied against the first (primary) alias in a repeater
 table (which is described below). There are four types of exceptions types
 that you can use, they are:

 - ignore

 The 'ignore' exception type causes Devmon to not display rows in a repeater
 table which have a primary oid that matches the exception regexp.

 - only

 The 'only' exception type causes Devmon to only display rows in a repeater
 table which have a primary oid that matches the exception regexp.

 - alarm

 The 'alarm' exception causes Devmon to only generate alarms for rows in a
 repeater table that have a primary oid that matches the exception regexp.

 - noalarm

 The 'noalarm' exception causes Devmon to not generate alarms for rows in a
 repeater table that have a primary oid that matches the exception regexp.

 The exceptions are applied in the order above, and one primary alias can
 match multiple exceptions. So if you have a primary alias that matches both
 an 'ignore' and an 'alarm' exception, no alarm will be generated (in fact,
 the row won't even be displayed in the repeater table).

 The example file listed above, from a cisco 2950 if_stat test, tells Devmon
 to only alarm on repeater table rows which have a primary oid (in this case,
 ifName) that starts with 'Gi' and has any number of characters after that
 (which will match any Gigabit interfaces on the switch). Also, it tells
 Devmon not to display any rows with a primary alias that has a value that
 behind with Nu (a Null interface) or Vl (A VLAN interface).


 The 'messages' file
 ------------------------------------------------------------------------------

 The messages file is what brings all the data collected from the other files
 in the template together in a single cohesive entry. It is basically a web
 page (indeed, you can add html to it, if you like) with some special macros
 embedded in it.

 An example of a simple messages file is as follows:

 -<start file>-----------------------------

    {upsStatus.errors}
    {upsBattStat.errors}
    {upsLoadOut.errors}
    {upsBattRep.errors}

    UPS status:

    Vendor:              apc
    Model:               {upsModel}

    UPS Status:          {upsOutStat}
    Battery Status:      {upsBattStat}

    Runtime Remaining:   {upsMinsRunTime} minutes
    Battery Capacity:    {upsBattCap}%
    UPS Load:            {upsLoadOut}%

    Voltage in:          {upsVoltageIn}v
    Voltage out:         {upsVoltageOut}v

    Last failure due to: {upsFailCause}
    Time on battery:     {upsSecsOnBatt} secs

 -<end file>-------------------------------

 You can see in this file that it is just a bunch of data aliases, with one or
 two special exceptions. Most of these will just be replaced with their
 corresponding values. You can see at the top of the file, however, that there
 are a few weird looking data aliases (the ones that end in .errors). These
 are just normal data aliases with a special flag appended to them, that lets
 Devmon know that you want something from them than just their data value.

 Here are all of the alias flags, and their functions:

 - color

 This flag will print out the bb/hobbit/xymon color string assigned to this
 data alias by the thresholds (this string looks like '&red' or '&green',
 etc). This color string will be interpreted by xymon as a colored icon, which
 makes alarm conditions much easier to recognize. Like the 'errors' flag, it
 will also modify the global color.

 - errors 

 The errors flag on a data alias will list any errors on this data alias. In
 this case, 'errors' refers to the message assigned to the alias from a non-
 green threshold match (the message is the value assigned in the fourth field
 of an entry in the thresholds file, remember?). If the value assigned to a
 data alias is green, then the value that replaces this flag will be blank.

 Error messages will always be printed at the TOP of the message file,
 regardless of where they are defined within it. This is done to make sure
 that the user sees any errors that might have occurred, which they might miss
 if the messages file is too long.

 The errors flag will also modify the global color of the message. So if this
 error flag reports a yellow error, and the global color is currently green,
 it will increase the global color to yellow. If the error flag reports a red
 error, it will increase the global color to red. The global color of a
 message defaults to green, and is modified upwards (if you consider more
 severe colors to be 'up') depending on the contents of the 'error' and
 'color' flags.
 
 Starting with the github version, we decide to 'propagate' errors and their
 messages from a transform to the next one. Why? 
 - There is a need to : tswitch was aleardy doing so 
 - We would like to catch also other errors and see were they come from 
 Errors generated by threshold are in fact just alarms. But we have also:
 - Connectivity error: Not any value or some "undefined" value -> Undef 
 - Computational error: Impossible result                      -> NaN
 - Impcompaible matrix: holes in a table                       -> n/a
 About the severity: 
 - During the threshold processing, the color(=severityi) and the error are set 
   - if yellow, red, clear: the error is set (and will be raised)
   - if green or blue: no error are raised
 - if error is set: furher transform will bypass the threshold processing 
 - Transform usually just transforms data, but if a problem is detected it can:
   - Set a severity (without setting the error), usually yellow (minor) 
   - The threshold processing is done, so the severity can be override
   - If not overriden, the error is set at the end of the threshold processing
 - A connectivity error is currently put in severity = 'gray' or 'no report'
 - A processing error is currenty put in 'yellow' and set the value to 'NaN'
 
 - msg

 The msg flag prints out the message assigned to the data alias by its
 threshold. Unlike the errors flag, it prints the message even if the data
 alias matches a green threshold and it also does NOT modify the global color
 of the message.

 - thresh

 The syntax for the thresh flag is {oid.thresh:<color>}. It displays the value
 in the threshold file (or custom threshold) that corresponds with the
 supplied color. So, {CPUTotal5Min.thresh:yellow} would display the template
 value for the yellow threshold for the CPUTotal5Min oid, or a per-device
 custom threshold if one was defined.

 A more complicated message file is this one, taken from a Cisco 2950 switch
 if_stat test:


 -<begin file------------------------------

      Ifc name|Ifc speed|Ifc status
      {ifName}{ifAliasBox}|{ifSpeed}|{ifStat.color}{ifStat}{ifStat.errors}

 -<end file>-------------------------------

 In this message file, we are using a repeater table. Repeater tables are used
 to display repeater-type data aliases (which ultimately stem from 'branch'
 type snmp oids). The 'TABLE:' keyword (case sensitive, no leading whitespace
 allowed) is what alerts Devmon that the next one to two lines are a repeater
 table definition.

 Devmon basically just builds an HTML table out of the repeater data. It can
 have an optional header, which should be specified on the line immediately
 after the 'TABLE:' tag. If no table header is desired, the line after the
 table tag should be the row data identifier.

 The row data identifier is the one that contains one or more data aliases.
 The first of these aliases is referred to as the 'primary' alias, and must be
 a repeater-type alias. Any other repeater type aliases in the row will be
 keyed off the primary alias; that is, if the primary aliases has leaves
 numbered '100,101,102,103,104', the table will have five rows, with the first
 row having all repeater aliases using leaf 100, the second row having all
 repeaters using leaf 101, etc. Any non-repeaters defined in the table will
 have a constant value throughout all of the rows.

 The TABLE: key can have one or more, comma-delimited options following it
 that allow you to modify the way in which Devmon will display the data. These
 options can have values assigned to them if they are not boolean ('nonhtml',
 for example, is boolean, while 'border' is not boolean).

 The TABLE options

 - nonhtml

 Don't use HTML tags when displaying the table. Instead all columns will
 be separated by a colon (:). This is useful for doing NCV rrd graphing
 in hobbit.

 - plain

 Don't do any formatting. This allows repeater data (each item on it's own
 line), without colons or HTML tables. One use of this option is to format
 repeater data with compatibility with a format Hobbit already understands. An
 example is available in the disk test for the linux-openwrt template.

 - noalarmsmsg

 Prevent Devmon from displaying the 'Alarming on' header at the top of a
 table.

 - alarmsonbottom

 Cause Devmon to display the 'Alarming on' message at the bottom of the table
 data, as opposed to the top.

 - border=n

 Set the HTML table border size that Devmon will use
 (a value of 0 will disable the border)

 - pad=n

 Set the HTML table cellpadding size that Devmon will use

 - rrd

 See the GRAPHING document in this directory for explanation

 An example of some TABLE options in use:

    TABLE: alarmsonbottom,border=0,pad=10

 The STATUS: key allows you to extend the first line of the status message
 that Devmon sends to BB/Hobbit/Xymon. For example, if you need to get data
 to a Xymon rrd collector module that evaluates data in the first line of the
 message (such as the Hobbit la collector which expects "up: <time>, %d
 users, %d procs load=%d.%d" you can use this key as follows to get a load
 average graph:

    STATUS: up: load={laLoadFloat2}


 Done!
 ------------------------------------------------------------------------------

 That's it! Once you've completed the five files mentioned above, you should,
 in theory, have a working template. I would recommend building the template
 under a separate 'test' installation of Devmon, as the single-node version
 of Devmon re-reads the template directory once per poll period, and having
 an incomplete or broken template will cause Devmon to throw error messages
 into its log.

 Try extracting the Devmon tarball to somewhere like "/usr/local/devmontest",
 and fiddle with the templates from there. Run Devmon from this directory in
 single-node mode, using a dummy bb-hosts file (even if your production Devmon
 cluster runs in multi-node mode, running the test Devmon in single node mode
 prevents you from having to create an additional database for your Devmon
 "test" installation). With the -vv and -p flags (i.e. devmon -vv -p), you
 will get verbose output from Devmon, and if you have a host in the bb-hosts
 file that matches the sysdesc in the specs file of the the model-vendor for
 the new template you created, you will also get textual output of your new
 template! (The -p flag causes Devmon to not run in the background and to
 print messages to STDOUT as opposed to sending them to the display server,
 and the -vv flag causes Devmon to log verbosely.)

 Once you are satisfied that your template is working correctly, you can put
 it to work in your production installation. In a single-node installation,
 this is as simple as copying the template directory to the appropriate
 subdirectory of your templates/ dir. On the next poll cycle, Devmon will pick
 up the new template, and any new hosts discovered by your readbbhosts cron
 job will be added to the Devmon database using this new template.

 In a multinode installation, adding a new template is only slightly more
 difficult. Copy the template directory to the appropriate place on the
 machine where you keep all your templates (earlier we recommended using your
 display server, and deleting all the template directories on the node
 machines). Once you have it in place, run devmon with the --synctemplates
 flag. This will read in the templates, update the database as necessary, and
 then notify all the Devmon nodes that they need to reload their templates. A
 full template reload on all your machines can take up to twice the interval
 of your polling cycle, so be patient!
