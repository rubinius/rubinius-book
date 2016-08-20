# Debugging the Platform

Sometimes flaws exist within the Rubinius Platform itself. Here are some guidelines, tips, and tricks for running Rubinius under a debugger to figure out the source of a bug.

## LLDB
The `lldb` debugger is part of the `LLVM` suite of tools. It's beyond the scope of this book to describe installation and configuration of these tools, so please google for `llvm installation` for your host OS for guidance.

### Attach To Crashed Process
If the runtime crashes with a SEGV or similar, the process will exit with a backtrace. In the old days a core file would be left behind that could be read by a debugger and used for determing the cause of the crash. Most modern OSes set the core file limit to 0 so no file gets written at all. 

Rubinius offers an alternative. It provides an environment variable along with a timeout (registered in minutes) that will pause the crashing process and provide time for the programmer to attach with a debugger. To enable this functionality, set the `RBX_PAUSE_ON_CRASH` to the number of minutes it should pause.

```
% RBX_PAUSE_ON_CRASH=60 rbx script.rb
```

Setting the variable to a zero or negative number will pause the process during a crash for 60 seconds before exiting. Using any positive integer and Rubinius will pause for that number of minutes before exiting.

### Run Specs Under LLDB
Occasionally the platform `specs` will uncover a bug in the runtime that causes a crash. Here's the command for running specs inside the debugger to catch the crash.

```
% cd /path/to/rubinius
% lldb -f bin/rbx -- mspec/bin/mspec-ci spec # can also use mspec/bin/mspec-run
(lldb) target create "bin/rbx"
Current executable set to 'bin/rbx' (x86_64).
(lldb) settings set -- target.run-args  "mspec/bin/mspec-ci"
(lldb) run
```

On some OSes it may prompt for a password to invoke the debugger.

### Ignore Signals
When executing a program under the debugger, the debugger defaults to breaking to a command prompt whenever a signal is encountered such as SIGVTALRM or SIGHUP. Oftentimes these signals are uninteresting and should be skipped or ignored by the debugger. To ignore them, execute the following:

```
* thread #3: tid = 0x1967650, 0x00007fff99057136 libsystem_kernel.dylib`__psynch_cvwait + 10, name = 'rbx.finalizer', stop reason = signal SIGHUP
    frame #0: 0x00007fff99057136 libsystem_kernel.dylib`__psynch_cvwait + 10
libsystem_kernel.dylib`__psynch_cvwait:
->  0x7fff99057136 <+10>: jae    0x7fff99057140            ; <+20>
    0x7fff99057138 <+12>: movq   %rax, %rdi
    0x7fff9905713b <+15>: jmp    0x7fff99052c53            ; cerror_nocancel
    0x7fff99057140 <+20>: retq   

(lldb) process handle -p true -s false SIGHUP
(lldb) c # to continue
```

### Set a Breakpoint on RubyException

```
(lldb) br set -n RubyException::raise
(lldb) run
```

This command sets a breakpoint in class `RubyException` at its `raise` method.

### Print a Ruby Backtrace
Inside the debugger the code is running in the context of the platform runtime. `lldb` itself doesn't know anything about the Ruby language. To get a Ruby backtrace from inside the debugger requires a little bit of work.

First, we must move to a frame that contains the `STATE` argument. The runtime uses `STATE` to reach all of its data structures. These structures are necessary for printing a backtrace. Attempting to print a Ruby backtrace from a frame without a `STATE` will fail because these structures will be unreachable. 

Moving up the stack to find a frame with a state is a manual process. Use the debugger command `up` to move up one frame at a time. At each new frame the debugger will print information regarding what file and line has been executed. In a separate window use a text editor to open each file and examine the function signature to see if it contains the `STATE` argument. If a frame with `STATE` was bypassed accidently, go back down the stack to it using the `down` command.

Upon finding a frame with the `STATE` argument, we can try to print a Ruby backtrace. There are two commands to try in the order given. The first command is preferred but sometimes it doesn't work. The second command almost always works however if the runtime state is corrupted it will likely crash the debugger.

```
(lldb) p __printbt__()
```

Alternately, try:

```
(lldb) p state->vm_->call_frame_->print_backtrace(state, 0, 0) # this might coredump the debugger
```

### Print a Ruby Object
Inside the runtime, most objects inherit from `Object` which is the parent class for all Ruby objects. With proper casting, the C++ Ruby object can be printed.

```
(lldb) p *(rubinius::Data*) obj
```

The trick is to find the correct class for the cast. Once again the programmer should have a separate window opened to the source tree. Use the backtrace information from the debugger to determine the point of execution in the code. The code reference should provide sufficient hints as to what objects are interesting and worthy of inspection.

### An Entire Debug Session Example
The following example is a debug session attempting to figure out why a Socket IO command was failing. The complete transcript is here showing all of the commands. If the sections above are insufficient to figure out how to execute a command, hopefully seeing them in the context of a real session will help clarify matters.

```
$ lldb -f bin/rbx -- socket.rb
(lldb) target create "bin/rbx"
Current executable set to 'bin/rbx' (x86_64).
(lldb) settings set -- target.run-args  "socket.rb"
(lldb) r
Process 17408 launched: '/source/rubinius/rubinius/bin/rbx' (x86_64)
#<File:0xf24 path=/dev/null>
#<UNIXSocket:fd 9>
#<UNIXSocket:fd 10>
An exception occurred running socket.rb

IO descriptor is not a Fixnum (RuntimeError)

Backtrace:

  Rubinius::CodeLoader#load_script at \
          core/code_loader.rb:505
  Rubinius::CodeLoader.load_script at \
          core/code_loader.rb:590
  Rubinius::Loader#script at \
          core/loader.rb:680
  Rubinius::Loader#main at \
          core/loader.rb:867
Process 17408 exited with status = 1 (0x00000001)
(lldb) br set -n RubyException::raise
Breakpoint 1: where = rbx`rubinius::RubyException::raise(rubinius::Exception*, bool) + 20 at exception.cpp:39, address = 0x0000000100053a74
(lldb) r
Process 17471 launched: '/source/rubinius/rubinius/bin/rbx' (x86_64)
#<File:0x2e8 path=/dev/null>
#<UNIXSocket:fd 9>
#<UNIXSocket:fd 10>
Process 17471 stopped
* thread #2: tid = 0xacd17, 0x0000000100053a74 rbx`rubinius::RubyException::raise(exception=0x0000000107182e18, make_backtrace=false) + 20 at exception.cpp:39, name = 'ruby.main', stop reason = breakpoint 1.1
    frame #0: 0x0000000100053a74 rbx`rubinius::RubyException::raise(exception=0x0000000107182e18, make_backtrace=false) + 20 at exception.cpp:39
   36  	  }
   37
   38  	  void RubyException::raise(Exception* exception, bool make_backtrace) {
-> 39  	    throw RubyException(exception, make_backtrace);
   40  	    // Not reached.
   41  	  }
   42
(lldb) up
frame #1: 0x000000010027e57e rbx`rubinius::Exception::raise_runtime_error(state=0x0000700000503c78, reason="IO descriptor is not a Fixnum") + 62 at exception.cpp:375
   372 	  }
   373
   374 	  void Exception::raise_runtime_error(STATE, const char* reason) {
-> 375 	    RubyException::raise(make_exception(state, get_runtime_error(state), reason));
   376 	  }
   377
   378 	  bool Exception::argument_error_p(STATE, Exception* exc) {
(lldb) down
frame #0: 0x0000000100053a74 rbx`rubinius::RubyException::raise(exception=0x0000000107182e18, make_backtrace=false) + 20 at exception.cpp:39
   36  	  }
   37
   38  	  void RubyException::raise(Exception* exception, bool make_backtrace) {
-> 39  	    throw RubyException(exception, make_backtrace);
   40  	    // Not reached.
   41  	  }
   42
(lldb) p exception
(rubinius::Exception *) $0 = 0x0000000107182e18
(lldb) up
frame #1: 0x000000010027e57e rbx`rubinius::Exception::raise_runtime_error(state=0x0000700000503c78, reason="IO descriptor is not a Fixnum") + 62 at exception.cpp:375
   372 	  }
   373
   374 	  void Exception::raise_runtime_error(STATE, const char* reason) {
-> 375 	    RubyException::raise(make_exception(state, get_runtime_error(state), reason));
   376 	  }
   377
   378 	  bool Exception::argument_error_p(STATE, Exception* exc) {
(lldb) p $0->show(state)
#<RuntimeError:0x107182e18>
  message: "IO descriptor is not a Fixnum"
  locations: nil
>
(rubinius::Object *) $1 = 0x000000000000001a
(lldb) up
frame #2: 0x0000000100296f7f rbx`rubinius::IO::descriptor(this=0x0000000107162848, state=0x0000700000503c78) + 159 at io.cpp:90
   87  	      return fd->to_native();
   88  	    }
   89
-> 90  	    Exception::raise_runtime_error(state, "IO descriptor is not a Fixnum");
   91  	  }
   92
   93  	  void IO::ensure_open(STATE) {
(lldb) p io_object->send(state, state->symbol("descriptor"))
error: no matching member function for call to 'send'
note: candidate function not viable: requires 3 arguments, but 2 were provided
note: candidate function not viable: requires 5 arguments, but 2 were provided
note: candidate function not viable: requires 5 arguments, but 2 were provided
error: 1 errors parsing expression
(lldb) p io_object->send(state, state->symbol("descriptor"), true)
(rubinius::Object *) $2 = 0x0000000000000000
(lldb) p state->vm_->thread_state_.current_exception()
(rubinius::Exception *) $3 = 0x0000000107183348
(lldb) p $3->show(state)
#<NoMethodError:0x107183348>
  message: "undefined method `descriptor' on an instance of Object."
  locations: #<Array:0x107184a20>
>
(rubinius::Object *) $4 = 0x000000000000001a
(lldb) p io_object->show(state)
#<Object:0x10305c688>
(rubinius::Object *) $5 = 0x000000000000001a
(lldb) p *$5
error: Couldn't apply expression side effects : Couldn't dematerialize a result variable: couldn't read its memory
(lldb) p *io_object
(rubinius::Object) $7 = {
  rubinius::ObjectHeader = {
    header = {
      f = {
        obj_type = ObjectType
        zone = MatureObjectZone
        age = 0
        meaning = eAuxWordEmpty
        Forwarded = 0
        Remember = 0
        Marked = 0
        InImmix = 1
        Pinned = 0
        Frozen = 0
        Tainted = 0
        Untrusted = 0
        LockContended = 0
        unused = 0
        aux_word = 0
      }
      flags64 = 4194621
    }
    klass_ = 0x000000010330f088
    ivars_ = 0x0000000107165f08
    __body__ = {}
  }
}
(lldb) p state->vm_->call_frame_->print_backtrace(state, 0, 0)
0x7000004f7cb0: Object#__script__ in /source/rubinius/rubinius/socket.rb:11 (+184)
0x7000004fa950: Rubinius::CodeLoader#load_script in core/code_loader.rb:505 (+52)
0x7000004fd610: Rubinius::CodeLoader.load_script in core/code_loader.rb:590 (+40)
0x7000005002d0: Rubinius::Loader#script in core/loader.rb:680 (+214)
0x700000502fa0: Rubinius::Loader#main in core/loader.rb:867 (+81)
(lldb) p state->vm_->get_ruby_frame(0)->self()
(rubinius::Object *) $8 = 0x000000010305c688
(lldb) p *$8
(rubinius::Object) $9 = {
  rubinius::ObjectHeader = {
    header = {
      f = {
        obj_type = ObjectType
        zone = MatureObjectZone
        age = 0
        meaning = eAuxWordEmpty
        Forwarded = 0
        Remember = 0
        Marked = 0
        InImmix = 1
        Pinned = 0
        Frozen = 0
        Tainted = 0
        Untrusted = 0
        LockContended = 0
        unused = 0
        aux_word = 0
      }
      flags64 = 4194621
    }
    klass_ = 0x000000010330f088
    ivars_ = 0x0000000107165f08
    __body__ = {}
  }
}
(lldb) p $8->ivars_
(rubinius::Object *) $10 = 0x0000000107165f08
(lldb) p $8->ivars_->show(state)
#<Rubinius::CompactLookupTable:0x107165f08>: 3
  :@file, :@client, :@server,
>
(rubinius::Object *) $11 = 0x000000000000001a
```
