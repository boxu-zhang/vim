`README.md` for boxu-zhang/vim

When I use vim-lldb plugin to create a target with 'Ltarget' command, it encountered a crash with SEGV signal which indicate there is a segment fault. And I google the reason, just a few comments and not really help. Most of the online answers tell that it should be a conflict between vim linked python and lldb linked python. Actually, it's just one piece of the puzzle. Another side is about the vim code, it has some internal bugs when deal with this kind of task. 

Let's have a deep look at the whole program. We have a vim program which have python built in, and a plugin script with lldb python script support. The lldb python script will reference to another C++ shared library called '\_lldb.so'. The insturction flows in a path: vim -> vim-lldb -> lldb python library -> \_lldb.so.

Moreover, there are several details that we need to knwo:
	1. The vim built in python can be static linked or dynamic linked, which you can specify with the --enable-pythoninterp=yes/dynamic.    	2. The vim configuration script will choose the first python binary in PATH variable to config the python library. If you have multiple pythons in your system then you must take care with this. 
	3. The \_lldb.so share library exports some functions(C/C++ functions operating the debugger) to python environment and provides several python script as the interface.
	4. The \_lldb.so also includes the python library in itself.

Here comes a question, what is gonna happen if there are two python library loaded in a single process? If you're interested, you can compile both lldb & vim with different configurations and see the result. To save some time, I just give the answer here: it give an exeception and cause vim failed to start up because the 'Virtual Machine Object' of python environment is broken. (This only happens when you use some auto load plugin requiring python support, in this case, it's lldb)

So, the first step we need to do is to make sure that this is only one python library loaded when we run vim. Achieve this is quite simple, use --enable-pythoninterp=dynamic when compile vim which enables dynamic linked python library of your vim binary. And don't worried about lldb side, it's already dynamic linked if you use system provided lldb. (I'm using the one xcode provided)

Next, it's the tricky/difficult part. After I googled some comments and done some experiments, I found it still not worked even if I've already make sure that there is only one python library load in runtime. Vim can start regular and load lldb python module correctly. But, once I set a target to debug with the vim plugin command 'Ltarget', it crashed again with the segment fault error. 

And from here, no google stuff helps. So, I have to dig by myself. Download the source code of lldb & vim, recompile them with debug information and it's very easy to pinpoint the crash location. However, it looks strange when it first come to me. The error location is a NULL pointer access which of cause leading to a segment fault. The location is near ScriptInterpreterPython.cpp:515 which is the real reason why there is a NULL pointer. At this line, lldb tries to convert a 'stdout' object to a PyFile_Type object. Usually, the stdout is provide by the 'python' application. Here, we are dealing with vim application but not 'python', and the 'stdout' object is actually set by vim application. The relative code reside in $vim_root/src/if_py_both.h and $vim_root/src/if_python.c. Through inspecting the two file, I found the type of 'stdout' object is actually a 'OutputType' python class. Vim implements a new class called 'OutputType' and use it as the 'stdout' object in the python runtime environment. As we mentioned before, the reason of why we get a NULL pointer in the crash location is that the code in 'ScriptInterpreterPython.cpp:515' tries to convert the 'stdout' object to PyFile_Type which is impossible because the real type of the object is 'OutputType'. And 'OutputType' has no relationship with PyFile_Type. 

As we know, a class cannot convert to another class if they are not related in parent/child. Since this, we make them parent/child. So, I modified the if_py_both.h & if_python.c to make 'OutputType' derives from 'PyFile_Type'. After the modification and recompiling of the code, everything works fine. 

All of the modification you can get from this git repo. Hope you like it.

References:(About the crash issue)
https://github.com/gilligan/vim-lldb/issues/12
https://github.com/gilligan/vim-lldb/issues/17

