# TransPoCer
transpocer

## Usage
Patch AFLplusplus first with our modifications
```sh
$ cp transpocer.patch.v1 AFLplusplus/
$ cd AFLplusplus/
$ git apply transpocer.patch.v1
```
Then compile fuzzer (with qemu and unicornafl mode)
```sh
$ make clean; make distrib -j100 
```

Prepare a new working directory and create the following intermediate files required for fuzzing: `transpocer.trace.targets`, `transpocer.template.gdb`, `transpocer.sim.csv`, and `transpocer.dict`.

Example of `transpocer.trace.targets` (a trace path consisting of function names separated by `,` without spaces):
```sh
$ cat transpocer.trace.targets
_start,__libc_start_main,main,[function_A],...,[function_Z],[vulnerable_function]
```
Example of `transpocer.template.gdb` (the placeholders `test_case_place_holder` and `trace_log_place_holder` will be automatically replaced during fuzzing):
```sh
$ cat transpocer.template.gdb
file [/path/to/binary]
set args [-xx -yy zz] test_case_place_holder

set follow-fork-mode parent
set height 0
shell rm -f tmp_current_bt
shell touch tmp_current_bt
set logging file tmp_current_bt 
set breakpoint pending on
set confirm off
# set gdb-workaround-stop-event disabled-deadlock

define save_backtrace
    ### save current stack trace
    set logging on
    bt
    set logging off
    ### count current number of lines
    shell wc -l tmp_current_bt | awk '{print $1}' > tmp_current_lines 2>/dev/null
    ### set $tmp_current_lines = `cat tmp_current_lines`
    shell echo "set \$tmp_current_lines=$(cat tmp_current_lines)" > tmp_gdb_var
    source tmp_gdb_var
    print $tmp_current_lines
    ### check if trace file exists
    shell if [ -f trace_log_place_holder ]; then echo 1; else echo 0; fi > tmp_file_exists 2>/dev/null
    ### set $tmp_file_exists = `cat tmp_file_exists`
    shell echo "set \$tmp_file_exists=$(cat tmp_file_exists)" > tmp_gdb_var
    source tmp_gdb_var
    if $tmp_file_exists == 1
        ### calculate lines
        shell wc -l trace_log_place_holder | awk '{print $1}' > tmp_file_lines 2>/dev/null
        ### set $tmp_file_lines = `cat tmp_file_lines`
        shell echo "set \$tmp_file_lines=$(cat tmp_file_lines)" > tmp_gdb_var
        source tmp_gdb_var
        print $tmp_file_lines
        ### compare and update to maintain the max stack trace 
        if $tmp_current_lines > $tmp_file_lines
            shell cp tmp_current_bt trace_log_place_holder 2>/dev/null
            echo "Updated stack trace (current: $tmp_current_lines > existing: $tmp_file_lines)\n"
        else
            echo "Skipped update (current: $tmp_current_lines <= existing: $tmp_file_lines)\n"
        end
    else
        print $tmp_file_exists
        ### create new trace_log_place_holder
        shell cp tmp_current_bt trace_log_place_holder 2>/dev/null
        echo "Created new stack trace file\n"
    end
    ### cleanup temp files
    shell rm -f tmp_current_bt tmp_current_lines tmp_file_exists tmp_file_lines tmp_gdb_var 2>/dev/null
    continue
end

### preload the library, enable complete breakpoints
start

tbreak [function_with_recurring_vulnerability]
commands
    save_backtrace
end

tbreak [function_in_target_trace]
commands
    save_backtrace
end

# start debugging
start
c
q
```
Example of `transpocer.sim.csv` (pairs to align the query with the original PoC trace and with the inferred trace): 
```sh
$ cat transpocer.sim.csv
##############################################################################
_start,_start,1.0
__libc_start_main,__libc_start_main,0.99
...
[function_in_PoC_trace],[function_in_target_trace],[similarity]
[function_in_vulnerable_binary],[function_in_target_binary],[similarity]
[function_in_target_binary],[function_in_target_binary],1.0
```
Example of `transpocer.dict` (key bytes extracted from original PoC, branches in vulnerable binary, and dynamic analysis):
```sh
$ cat transpocer.dict 
"\x4d\x5a"
"\x4e\x45"
"\x4c\x01"
"\x6a\x01\x58\xc2\x0c"
"\x02\x21\x0b\x01"
"\xdd"
"\xff\xe1\xff"
"\xe1"
"\x02"
"\x30\x03\x04"
"\x04"
"\x39\x39\x39\x39\x39\x39"
"\x01"
"\x73\x1f\x45\x58\x45\x20\x58\x69\x74\x68\x20\x5a\x4d"
```

Run commands (AFL++ with transpocer strategy)
```sh
$ /path/to/afl-fuzz -p transpocer -t 10000 -i /path/to/in -o /path/to/out -x /path/to/workdir/transpocer.dict -- /path/to/target/binary -xx -yy  @@ 
# or qemu mode
$ /path/to/afl-fuzz -p transpocer -Q -t 10000 -i /path/to/in -o /path/to/out -x /path/to/workdir/transpocer.dict -- /path/to/target/binary -xx -yy  @@ 

### main fuzzing

 AFL ++4.33a {default} (...xxxxxxxxxxxxxx/path/to/target/binary) [transpocer]
┌─ process timing ────────────────────────────────────┬─ overall results ────┐
│        run time : 0 days, 0 hrs, 0 min, 10 sec      │  cycles done : 0     │
│   last new find : 0 days, 0 hrs, 0 min, 0 sec       │ corpus count : 27    │
│last saved crash : none seen yet                     │saved crashes : 0     │
│ last saved hang : none seen yet                     │  saved hangs : 0     │
├─ cycle progress ─────────────────────┬─ map coverage┴──────────────────────┤
│  now processing : 0.0 (0.0%)         │    map density : 2.62% / 2.80%      │
│  runs timed out : 0 (0.00%)          │ count coverage : 1.24 bits/tuple    │
├─ stage progress ─────────────────────┼─ findings in depth ─────────────────┤
│  now trying : bitflip 2/1            │ favored items : 1 (3.70%)           │
│ stage execs : 7/15 (46.67%)          │  new edges on : 23 (85.19%)         │
│ total execs : 217                    │ total crashes : 0 (0 saved)         │
│  exec speed : 20.63/sec (slow!)      │  total tmouts : 0 (0 saved)         │
├─ fuzzing strategy yields ────────────┴─────────────┬─ item geometry ───────┤
│   bit flips : 16/16, 0/0, 0/0                      │    levels : 2         │
│  byte flips : 0/0, 0/0, 0/0                        │   pending : 27        │
│ arithmetics : 0/0, 0/0, 0/0                        │  pend fav : 1         │
│  known ints : 0/0, 0/0, 0/0                        │ own finds : 25        │
│  dictionary : 0/0, 0/0, 0/0, 0/0                   │  imported : 0         │
│havoc/splice : 0/0, 0/0                             │ stability : 100.00%   │
│py/custom/rq : unused, unused, unused, unused       ├───────────────────────┘
│    trim/eff : n/a, n/a                             │          [cpu000:  5%]
└─ strategy: explore ────────── state: started :-) ──┘
     ┌───────  [transpocer note] in main process, status : 0 ─────────┐

### gdb backtracing

 AFL ++4.33a {default} (...xxxxxxxxxxxxxx/path/to/target/binary) [transpocer]
┌─ process timing ────────────────────────────────────┬─ overall results ────┐
│        run time : 0 days, 0 hrs, 0 min, 10 sec      │  cycles done : 0     │
│   last new find : 0 days, 0 hrs, 0 min, 0 sec       │ corpus count : 27    │
│last saved crash : none seen yet                     │saved crashes : 0     │
│ last saved hang : none seen yet                     │  saved hangs : 0     │
├─ cycle progress ─────────────────────┬─ map coverage┴──────────────────────┤
│  now processing : 0.0 (0.0%)         │    map density : 2.62% / 2.80%      │
│  runs timed out : 0 (0.00%)          │ count coverage : 1.24 bits/tuple    │
├─ stage progress ─────────────────────┼─ findings in depth ─────────────────┤
│  now trying : bitflip 2/1            │ favored items : 1 (3.70%)           │
│ stage execs : 7/15 (46.67%)          │  new edges on : 23 (85.19%)         │
│ total execs : 217                    │ total crashes : 0 (0 saved)         │
│  exec speed : 20.63/sec (slow!)      │  total tmouts : 0 (0 saved)         │
├─ fuzzing strategy yields ────────────┴─────────────┬─ item geometry ───────┤
│   bit flips : 16/16, 0/0, 0/0                      │    levels : 2         │
│  byte flips : 0/0, 0/0, 0/0                        │   pending : 27        │
│ arithmetics : 0/0, 0/0, 0/0                        │  pend fav : 1         │
│  known ints : 0/0, 0/0, 0/0                        │ own finds : 25        │
│  dictionary : 0/0, 0/0, 0/0, 0/0                   │  imported : 0         │
│havoc/splice : 0/0, 0/0                             │ stability : 100.00%   │
│py/custom/rq : unused, unused, unused, unused       ├───────────────────────┘
│    trim/eff : n/a, n/a                             │          [cpu000:  5%]
└─ strategy: explore ────────── state: started :-) ──┘
     ┌───────  [transpocer note] in gdb process, status : 1 ──────────┐

```