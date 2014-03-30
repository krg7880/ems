# Extended Memory Semantics (EMS)

EMS is a Node.js NPM Package that makes possible shared memory multithreaded parallelism.

### <em>EMS is targeted at problems too large for one core, but too small for a scalable cluster.</em>

A modern multicore server has 16-32 cores and over 200GB of memory,
equivalent to an entire rack of systems from a few years ago.
As a consequence, jobs requiring a cluster of servers running Map-Reduce 5 years ago 
can now be performed entirely in shared memory on a single server
without using distributed programming.


## Types of Concurrency
<center>
  <table >
    <tr>
      <td>
EMS complements an application's 
asynchronous and distributed concurrency with
multithreading, transactional memory, and 
other fine-grained synchronization capabilities.
<br><br>
EMS supports Fork-Join and Bulk Synchronous Parallel (BSP) execution models.
Fork-Join execution begins with a single thread that creates and ends
parallel regions as needed, whereas BSP execution begins with each
thread entering the program at <code>main()</code> and executing all
the statements.
<br><br>
The same underlying EMS primitives (stacks, queues, transactions, 
atomic read-modify-write operations, etc.)
are used in both execution modes.
            </td>
            <td width="50%">
              <center>
  <center>
  <img style="height:350px; margin-left: 80px;" src="http://synsem.com/EMS.js/typesOfParallelism.svg" type="image/svg+xml"  />
  </center>
              </center>
            </td>
    </tr>
  </table>
</center>


## Examples
### Word Count using Atomic Operations
The standard example for Map-Reduce is word counting, this program shows EMS
executing a loop in parallel and atomically incrementing the count of each
word found.

```javascript
var ems  = require('ems')(process.argv[2])  // Command line argument for # of threads to use

//  Allocate the EMS shared memory holding the word count dictionary
var maxNKeys    = 10000000
var wordCounts   = ems.new( {
    dimensions : [ maxNKeys ],  // Maximum # of different keys the array can store
    heapSize   : maxNKeys * 10, // 10 bytes of storage per key, used to store dictionary word itself
    useMap     : true,          // Use a key-index mapping, not integer indexes
    setFEtags  : 'full',        // Initial full/empty state of array elements
    dataFill   : 0              // Initial value of new keys
} )

//  Get the list of files to process
var dir          = fs.readdirSync('/Data/Gutenberg/all/');
var splitPattern = new RegExp(/[ \n,\.\\/_\-\<\>:\;\!\@\#\$\%\&\*\(\)=\[\]|\"\'\{\}\?\—]/)

//  Parallel loop over documents
ems.parForEach(0, dir.length,  function(docNum) {
    var text = fs.readFileSync('/Data/Gutenberg/all/' + dir[bufNum], 'utf8', "r")    
    var words = text.replace(/[\n\r]/g,' ').toLowerCase().split(splitPattern)
    words.forEach( function(word, wordN) {
        wordCounts.faa(word, 1)  // Atomic Fetch-and-Add updates word count
    } )
} )
```

### Transactional Memory
An example of using EMS Transactional Memory to atomically update
two account balances while simultaneously preventing updates to the user's 
customer records.

```javascript
var ems = require('ems')(process.argv[2])        // Initialize EMS
var customers = ems.new(...)                     // Allocate EMS memory for customer records
var accounts  = ems.new(...)                     // Allocate accounts
...
// Start a transaction involving Bob and Sue
var transaction= ems.tmStart( [ [customers, 'Bob Smith', true],  // Read-only:  Bob's customer record
                                [customers, 'Sue Jones', true],  // Read-only:  Sue's customer record
                                [accounts, 'Bob Smith'],         // Read-Write: Bob's account balance
                                [accounts, 'Sue Jones'] ] )      // Read-Write: Sue's account balance
                                
// Transfer the payment and update the balances
var bobsBalance = accounts.read('Bob Smith')                // Read the balance of Bob's account
accounts.write('Bob Smith', bobsBalance - paymentAmount)    // Deduct the payment and write the new balance back
var suesBalance = accounts.read('Sue Jones')                // Read the balance of Sue's account
accounts.write('Sue Jones', suesBalance + paymentAmount)    // Add the payment to Sue's account

// Commit the transaction or abort it if NSF
if(balance > paymentAmount) {                               // Test for overdraft
    ems.tmEnd(transaction, true)                            // Sufficient funds, commit transaction
} else {
    ems.tmEnd(transaction, false)                           // Not Sufficient Funds, roll back transaction
}
```


<br><br>
## Synchronization as a Property of the Data, Not a Duty for Tasks

EMS internally stores tags that are used for synchronization of
user data, allowing synchronization to happen independently of
the number or kind of processes accessing the data.  The tags
can be thought of as being in one of three states, <em>Empty,
Full,</em> or <em>Read-Only</em>, and the EMS intrinsic functions
enforce atomic access through automatic state transitions.

The EMS array may be indexed directly using an integer, or using a key-index
mapping from any primitive type.  When a map is used, the key and data
  itself are updated atomically.
  

  <center>
  <table >
    <tr>
      <td>
    <center>
      <img style="width:350px; "
	   src="http://synsem.com/EMS.js/memLayoutLogical.svg" type="image/svg+xml" />
      <em>    <br><br> 
    EMS memory is an array of JSON primitive values
        (Number, Boolean, String, or Undefined) accessed using atomic
        operators and/or transactional memory.  Safe parallel access
        is managed by passing through multiple gates: First mapping a
        key to an index, then accessing user data protected by EMS
        tags, and completing the whole operation atomically.
      </center>
    </em>
    </td>
    <td width="50%">
      <center>
  <img style="height:270px; "
   src="http://synsem.com/EMS.js/fsmSimple.svg" type="image/svg+xml" />
    <em>    <br><br> EMS Data Tag Transitions & Atomic operations:
    F=Full, E=Empty, X=Don't Care, RW=Readers-Writer lock (# of current readers)
    CAS=Compare-and-Swap, FAA=Fetch-and-Add</em>
      </center>
    </td>
    </tr>
  </table>
  </center>  

### Less is More
Because all systems are already multicore, 
multithreading requires no additional equipment, system permissions,
or application services, making it easy to get started.
The reduced complexity of
lightweight threads communicating through shared memory
is reflected in a rapid code-debug cycle for ad-hoc application development.


## More Technical Information

For a more complete description of the principles of operation,
[visit the EMS web site.](http://synsem.com/EMS.js/)
  
<br>
<center>
  <img src="http://synsem.com/EMS.js/blockDiagram.svg" type="image/svg+xml" height="300px" style="vertical-align:text-top;"/>
</center>

<br>

## Download
#### NPM Package
EMS is available as a NPM Package.  It has no external dependencies,
but does require compiling native C++ functions using <code>node-gyp</code>,
which is also available as a NPM.
<br>
```csh
npm -g install ems
```
Note: Install EMS globally (<code>-g</code>) so forked threads can find the package.


#### GitHub
Download the source code.  Be sure to compile the native code with <code>node-gyp</code>,
```csh
git clone https://github.com/SyntheticSemantics/ems.git
cd ems/Addon/
node-gyp configure
node-gyp build
cd ../Examples/
node streams.js 8
```
Note that because EMS fortks threads that also use the EMS module, 
developers modifying EMS source code must ensure the <code>emsThreadStub</code>
is set to use the development EMS package, not the one installed system-wide,
which will be used by default.


## Platforms Supported
Presently Mac/Darwin and Linux are supported.  There is no technical reason it would not 
also work on Windows with POSIX functions replaced using the MS equivalent.
Contributions are welcomed!

## License
BSD, other commercial and open source licenses are available.

## Links
[Visit the EMS web site.](http://synsem.com/EMS.js/)

## About
<img src="http://synsem.com/synsem_logo_black.svg" type="image/svg+xml" height="30px" style="vertical-align:middle;"> [SynSem](http://synsem.com) provides tools and services for shared memory parallelism, 
GPU/FPGA/DSP/CPU hybrid computing, high performance computing, and hardware/software co-design.