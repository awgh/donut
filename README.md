# Using Donut

![Alt text](https://github.com/TheWover/donut/blob/master/img/donut.PNG?raw=true "An ASCII donut")                                                                                                               

Version: 0.9.2 *please submit issues and requests for v1.0 release*

Odzhan's blog post (about the generator): https://modexp.wordpress.com/2019/05/10/dotnet-loader-shellcode/

TheWover's blog post (detailed walkthrough, and about how donut affects tradecraft): https://thewover.github.io/Introducing-Donut/

v0.9.2 release blog post: https://thewover.github.io/Bear-Claw/

## Introduction

Donut generates x86 or x64 shellcode from VBScript, JScript, EXE, DLL (including .NET Assemblies) files. This shellcode can be injected into an arbitrary Windows processes for in-memory execution. Given a supported file type, parameters and an entry point where applicable (such as Program.Main), it produces position-independent shellcode that loads and runs entirely from memory. A module created by donut can either be staged from a URL or stageless by being embedded directly in the shellcode. Either way, the module is encrypted with the Chaskey block cipher and a 128-bit randomly generated key. After the file is loaded through the PE/ActiveScript/CLR loader, the original reference is erased from memory to deter memory scanners. For .NET Assemblies, they are loaded into a new Application Domain to allow for running Assemblies in disposable AppDomains.

It can be used in several ways.

## As a Standalone Tool

Donut can be used as-is to generate shellcode from VBS/JS/EXE/DLL files or .NET Assemblies. A Linux and Windows executable and a Python module are provided for loader generation. The Python documentation can be found [here](https://github.com/TheWover/donut/blob/master/docs/2019-08-21-Python_Extension.md). The command-line syntax is as described below.

```
 usage: donut [options] -f <EXE/DLL/VBS/JS>

                   -MODULE OPTIONS-

       -f <path>            .NET assembly, EXE, DLL, VBS, JS file to execute in-memory.
       -n <name>            Module name. Randomly generated by default.
       -u <URL>             HTTP server that will host the donut module.

                   -PIC/SHELLCODE OPTIONS-

       -a <arch>            Target architecture : 1=x86, 2=amd64, 3=amd64+x86(default).
       -b <level>           Bypass AMSI/WLDP : 1=skip, 2=abort on fail, 3=continue on fail.(default)
       -o <loader>          Output file. Default is "loader.bin"
       -e                   Encode output file with Base64. (Will be copied to clipboard on Windows)
       -t                   Run entrypoint for unmanaged EXE as a new thread. (replaces ExitProcess with ExitThread in IAT)
       -x                   Call RtlExitUserProcess to terminate the host process. (RtlExitUserThread is called by default)

                   -DOTNET OPTIONS-

       -c <namespace.class> Optional class name.  (required for .NET DLL)
       -m <method | api>    Optional method or API name for DLL. (a method is required for .NET DLL)
       -p <parameters>      Optional parameters inside quotations.
       -r <version>         CLR runtime version. MetaHeader used by default or v4.0.30319 if none available.
       -d <name>            AppDomain name to create for .NET. Randomly generated by default.

 examples:

    donut -f c2.dll
    donut -a1 -cTestClass -mRunProcess -pnotepad.exe -floader.dll
    donut -f loader.dll -c TestClass -m RunProcess -p"calc notepad" -u http://remote_server.com/modules/
```

### Building Donut

Tags have been provided for each release version of donut that contain the compiled executables. 

* v0.9.2, Bear Claw: https://github.com/TheWover/donut/releases/tag/v0.9.2
* v0.9.2 Beta: https://github.com/TheWover/donut/releases/tag/v0.9.2
* v0.9.1, Apple Fritter: https://github.com/TheWover/donut/releases/tag/v0.9.1
* v0.9, Initial Release: https://github.com/TheWover/donut/releases/tag/v0.9

However, you may also clone and build the source yourself using the provided makefiles. 

## Building From Repository

From a Windows command prompt or Linux terminal, clone the repository and change to the donut directory.

```
git clone http://github.com/thewover/donut
cd donut
```

## Linux

Simply run make to generate an executable, static and dynamic libraries.

```
make
make clean
make debug
```

## Windows

Start a Microsoft Visual Studio Developer Command Prompt and `` cd `` to donut's directory. The Microsft (non-gcc) Makefile can be specified with ``` -f Makefile.msvc ```. The makefile provides the following commmands to build donut:

```
nmake -f Makefile.msvc
nmake clean -f Makefile.msvc
nmake debug -f Makefile.msvc
```

## As a Library

donut can be compiled as both dynamic and static libraries for both Linux (*.a* / *.so*) and Windows(*.lib* / *.dll*). It has a simple API that is described in *docs/api.html*. Two exported functions are provided: ``` int DonutCreate(PDONUT_CONFIG c) ``` and ``` int DonutDelete(PDONUT_CONFIG c) ``` .

## As a Python Module

Donut can be installed and used as a Python module. To install Donut from your current directory, use pip for Python3.

```
pip install .
```

Otherwise, you may install Donut as a Python module by grabbing it from the PyPi repostiory.

```
pip install donut-shellcode
```

## As a Template - Rebuilding the shellcode

*loader/* contains the in-memory execution code for EXE/DLL/VBS/JS and .NET assemblies, which should successfully compile with both Microsoft Visual Studio and MinGW-w64. Make files have been provided for both compilers. Whenever files in the loader directory have been changed, recompiling for all architectures is recommended before rebuilding donut.

### Microsoft Visual Studio

**Due to recent changes in the MSVC compiler, we now only support MSVC versions 2019 and later.**

Open the x64 Microsoft Visual Studio build environment, switch to the *loader* directory, and type the following:

```
nmake clean -f Makefile.msvc
nmake -f Makefile.msvc
```

This should generate a 64-bit executable (*loader.exe*) from *loader.c*. exe2h will then extract the shellcode from the *.text* segment of the PE file and save it as a C array to *loader_exe_x64.h*. When donut is rebuilt, this new shellcode will be used for all loaders that it generates.

To generate 32-bit shellcode, open the x86 Microsoft Visual Studio build environment, switch to the loader directory, and type the following:

```
nmake clean -f Makefile.msvc
nmake x86 -f Makefile.msvc
```

This will save the shellcode as a C array to *loader_exe_x86.h*.

### MinGW-W64

Assuming you're on Linux and *MinGW-W64* has been installed from packages or source, you may still rebuild the shellcode using our provided makefile. Change to the *loader* directory and type the following:

```
make clean -f Makefile.mingw
make -f Makefile.mingw
```

Once you've recompiled for all architectures, you may rebuild donut.

## Bypasses

Donut includes a bypass system for AMSI and other security features. Currently we bypass:

* AMSI in .NET v4.8
* Device Guard policy preventing dynamicly generated code from executing

You may customize our bypasses or add your own. The bypass logic is defined in loader/bypass.c.

Each bypass implements the DisableAMSI fuction with the signature ```BOOL DisableAMSI(PDONUT_INSTANCE inst)```, and comes with a corresponding preprocessor directive. We have several ```#if defined``` blocks that check for definitions. Each block implements the same bypass function. For instance, our first bypass is called ```BYPASS_AMSI_A```. If donut is built with that variable defined, then that bypass will be used.

Why do it this way? Because it means that only the bypass you are using is built into loader.exe. As a result, the others are not included in your shellcode. This reduces the size and complexity of your shellcode, adds modularity to the design, and ensures that scanners cannot find suspicious blocks in your shellcode that you are not actually using.

Another benefit of this design is that you may write your own AMSI bypass. To build Donut with your new bypass, use an ```if defined``` block for your bypass and modify the makefile to add an option that builds with the name of your bypass defined.

If you wanted to, you could extend our bypass system to add in other pre-execution logic that runs before your .NET Assembly is loaded. 

Odzhan wrote a [blog post](https://modexp.wordpress.com/2019/06/03/disable-amsi-wldp-dotnet/) on the details of our AMSI bypass research.

### Additional features.

These are left as exercises to the reader. I would personally recommend:

* Add environmental keying
* Make donut polymorphic by obfuscating *loader* every time shellcode is generated
* Integrate donut as a module into your favorite RAT/C2 Framework

## Disclaimers

* No, we will not update donut to counter signatures or detections by any AV.
* We are not responsible for any misuse of this software or technique. Donut is provided as a demonstration of CLR Injection through shellcode in order to provide red teamers a way to emulate adversaries and defenders a frame of reference for building analytics and mitigations. This inevitably runs the risk of malware authors and threat actors misusing it. However, we believe that the net benefit outweighs the risk. Hopefully that is correct.

# How it works

## Procedure for Assemblies

Donut uses the Unmanaged CLR Hosting API to load the Common Language Runtime. If necessary, the Assembly is downloaded into memory. Either way, it is decrypted using the Chaskey block cipher. Once the CLR is loaded into the host process, a new AppDomain will be created using a random name unless otherwise specified. Once the AppDomain is ready, the .NET Assembly is loaded through AppDomain.Load_3. Finally, the Entry Point specified by the user is invoked with any specified parameters.

The logic above describes how the shellcode generated by donut works. That logic is defined in *loader.exe*. To get the shellcode, *exe2h* extracts the compiled machine code from the *.text* segment in *loader.exe* and saves it as a C array to a C header file. *donut* combines the shellcode with a Donut Instance (a configuration for the shellcode) and a Donut Module (a structure containing the .NET assembly, class name, method name and any parameters).

Refer to MSDN for documentation on the Undocumented CLR Hosting API: https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/hosting/clr-hosting-interfaces

For a standalone example of a CLR Host, refer to Casey Smith's AssemblyLoader repo: https://github.com/caseysmithrc/AssemblyLoader

Detailed blog posts about how donut works are available at both Odzhan's and TheWover's blogs. Links are at the top of the README.

## Procedure for ActiveScript/XSL

The details of how Donut loads scripts and XSL files from memory have been detailed by Odzhan in a [blog post](https://modexp.wordpress.com/2019/07/21/inmem-exec-script/).

## Procedure for PE Loading

The details of how Donut loads PE files from memory have been detailed by Odzhan in a [blog post](https://modexp.wordpress.com/2019/06/24/inmem-exec-dll/).

Only PE files with relocation information (.reloc) are supported. TLS callbacks are only executed upon process creation.

## Components

Donut contains the following elements:

* donut.c: The source code for the donut loader generator.
* donut.exe: The compiled loader generator as an EXE.
* donut.py: The donut loader generator as a Python script *(planned for version 1.0)*
* donutmodule.c: The CPython wrapper for Donut. Used by the Python module.
* setup.py: The setup file for installing Donut as a Pip Python3 module.
* lib/donut.dll, lib/donut.lib: Donut as a dynamic and static library for use in other projects on Windows platform.
* lib/donut.so, lib/donut.a: Donut as a dynamic and static library for use in other projects on the Linux platform.
* lib/donut.h: Header file to include if using the static or dynamic libraries in a C/C++ project.
* loader/loader.c: Main file for the shellcode.
* loader/inmem_dotnet.c: In-Memory loader for .NET EXE/DLL assemblies.
* loader/inmem_pe.c: In-Memory loader for EXE/DLL files.
* loader/inmem_script.c: In-Memory loader for VBScript/JScript files.
* loader/activescript.c: ActiveScriptSite interface required for in-memory execution of VBS/JS files.
* loader/wscript.c: Supports a number of WScript methods that cscript/wscript support.
* loader/bypass.c: Functions to bypass Anti-malware Scan Interface (AMSI) and Windows Local Device Policy (WLDP).
* loader/http_client.c: Downloads a module from remote staging server into memory.
* loader/peb.c: Used to resolve the address of DLL functions via Process Environment Block (PEB).
* loader/clib.c: Replaces common C library functions like memcmp, memcpy and memset.
* loader/inject.exe: The compiled C shellcode injector.
* loader/inject.c: A C shellcode injector that injects loader.bin into a specified process for testing.
* loader/runsc.c: A C shellcode runner for testing loader.bin in the simplest manner possible.
* loader/runsc.exe: The compiled C shellcode runner.
* loader/exe2h/exe2h.c: Source code for exe2h.
* loader/exe2h/exe2h.exe: Extracts the useful machine code from loader.exe and saves as array to C header file.
* encrypt.c: Chaskey 128-bit block cipher in Counter (CTR) mode used for encryption.
* hash.c: Maru hash function. Uses the Speck 64-bit block cipher with Davies-Meyer construction for API hashing.

# Subprojects

There are three companion projects provided with donut:

* DemoCreateProcess: A sample .NET Assembly to use in testing. Takes two command-line parameters that each specify a program to execute.
* DonutTest: A simple C# shellcode injector to use in testing donut. The shellcode must be base64 encoded and copied in as a string. 
* ModuleMonitor: A proof-of-concept tool that detects CLR injection as it is done by tools such as donut and Cobalt Strike's execute-assembly.
* ProcessManager: A Process Discovery tool that offensive operators may use to determine what to inject into and defensive operators may use to determine what is running, what properties those processes have, and whether or not they have the CLR loaded. 

# Project plan

* ~~Create a donut Python C extension that allows users to write Python programs that can use the donut API programmatically. It would be written in C, but exposed as a Python module.~~
* Create a C# version of the generator.
* Create a donut.py generator that uses the same command-line parameters as donut.exe.
* Add support for HTTP proxies.
~~* Find ways to simplify the shellcode if possible.~~
* Write a blog post on how to integrate donut into your tooling, debug it, customize it, and design loaders that work with it.
* ~~Dynamic Calls to DLL functions.~~
* Handle the ProcessExit event from AppDomain using unmanaged code.
