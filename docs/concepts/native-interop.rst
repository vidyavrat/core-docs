Native interoperability
=======================

In this document we will dive a little bit deeper into all three ways of doing 
"native interoperability" that are available on the .NET platform. 

There are a few of reasons why you would want to call into native code:

* Operating Systems come with a large volume of APIs that are 
  not present in the managed class libraries. A prime example for this would be 
  access to hardware or operating system management functions.
* Communicating with other components that have or can produce C-style ABIs 
  (native ABIs). This covers, for example, Java code that is exposed via 
  `Java Native Interface (JNI) <http://docs.oracle.com/javase/8/docs/technotes/guides/jni/>`_ 
  or any other managed language that could produce a native component. 
* On Windows, most of the software that gets installed, such as Microsoft Office
  suite, registers COM components that represent their programs and allow developers 
  to automate them or use them. This also requires native interoperability.

Of course, the list above does not cover all of the potential situations and 
scenarios in which the developer would want/like/need to interface with native 
components. .NET class library, for instance, uses the native interoperability 
support to implement a fair number of its APIs, like console support and 
manipulation, file system access and others. However, it is important to note 
that there is an option, should one need it.

.. note::
    Most of the examples in this document will be presented for all three 
    supported platforms for .NET Core (Windows, Linux, OS X). However, for some 
    short and illustrative examples, I have decided to leave just one sample 
    that will use Windows filenames and extensions (i.e. "dll" for libraries). 
    This does not mean that those features are not available on Linux or OS X, 
    it was done merely for conveince sake. 
    
Platform Invoke (P/Invoke)
--------------------------
P/Invoke is a technology that allows you to access structs, callbacks and functions
in unmanaged libraries from your managed code. Most of the P/Invoke API is contained 
in two namespaces: ``System`` and ``System.Runtime.InteropServices``. Using these 
two namespaces will allow you access to the attributes that describe how you 
want to communicate with the native component. 

Let's start from the most common example, and that is calling unmanaged functions 
in your managed code. Let's show a message box from a command-line application:

.. code-block:: c#
    :linenos:
    
    using System.Runtime.InteropServices;

    public class Program {
    
        [DllImport("user32.dll")]
        public static extrn int MessageBox(IntPtr hWnd, String text, String caption, int options);

        public static void Main(string[] args) {
            MessageBox(IntPtr.Zero, "Command-line message box", "Attention!", 0);
        }
    }

The example above is pretty simple, but it does show off what is needed to 
invoke unmanaged functions from managed code. Let's step through the example:

* Line #1 shows the using statement for the ``System.Runtime.InteropServices`` 
  which is the namespace that holds all of the items we need. 
* Line #5 introduces the ``DllImport`` attribute. This attribute is crucial, as 
  it tells the runtime that it should load the unmanaged DLL. This is the DLL 
  into which we wish to invoke.
* Line #6 is the crux of the P/Invoke work. It defines a managed method that has
  the **exact same signature** as the unmanaged one. The declaration has a new 
  keyword that you can notice, ``extern``, which tells the runtime this is an 
  external method, and that when you invoke it, the runtime should find it in the 
  DLL specified in ``DllImport`` attribute.

The rest of the example is just invoking the method as you would any other managed 
method. 

The sample is similar for OS X. One thing that needs to change is, of course, 
the name of the library in the ``DllImport`` attribute, as OS X has a different 
scheme of naming dynamic libraries. The sample below uses the ``getpid(2)`` 
function to get the process ID of the application and print it out to the console. 

.. code-block:: c#
    :linenos: 

    using System;
    using System.Runtime.InteropServices;

    namespace PInvokeSamples {
        public static class Program {
            
            [DllImport("libSystem.dylib")]
            private static extern int getpid();

            public static void Main(string[] args){
                int pid = getpid();
                Console.WriteLine(pid);
            }
        }
    }

It is similar on Linux, of course. The function name is same, since ``getpid(2)`` 
is `POSIX <https://en.wikipedia.org/wiki/POSIX>`_ system call. 

.. code-block:: c#
    :linenos:

    using System;
    using System.Runtime.InteropServices;

    namespace PInvokeSamples {
        public static class Program {
            
            [DllImport("libc.so")]
            private static extern int getpid();

            public static void Main(string[] args){
                int pid = getpid();
                Console.WriteLine(pid);
            }
        }
    }

Invoking managed code from unmanaged code
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
Of course, the runtime allows communication to flow both ways 
which enables you to call into managed artifacts from native functions, using 
function pointers. The closest thing to a function pointer in managed code is a 
**delegate**, so this is what is used to allow callbacks from native code into 
managed code. 

The way to use this feature is similar to managed to native process described 
above. For a given callback, you define a delegate that matches the signature, 
and pass that into the external method. The runtime will take care of everything 
else.

.. code-block:: c#
    :linenos:

    using System;
    using System.Runtime.InteropServices;

    namespace ConsoleApplication1 {

        class Program {

            delegate bool EnumWC(IntPtr hwnd, IntPtr lParam);

            [DllImport("user32.dll")]
            static extern int EnumWindows(EnumWC hWnd, IntPtr lParam);

            static bool OutputWindow(IntPtr hwnd, IntPtr lParam) {
                Console.WriteLine(hwnd.ToInt64());
                return true;
            }

            static void Main(string[] args) {
                EnumWindows(OutputWindow, IntPtr.Zero);
            }
        }
    }
    

Before we walk through our example, it is good to go over the signatures of the 
unmanaged functions we need to work with. The function we want to call to 
enumerate all of the windows has the following signature:
``BOOL EnumWindows (WNDENUMPROC lpEnumFunc, LPARAM lParam);``

The first parameter is a callback. The said callback has the following signature: 
``BOOL CALLBACK EnumWindowsProc (HWND hwnd, LPARAM lParam);``

With this in mind, let's walk through the example:

* Line #8 in the example defines a delegate that matches the signature of the 
  callback from unmanaged code. Notice how the LPARAM and HWND types are 
  represented using ``IntPtr`` in the managed code. 
* Lines #10 and #11 introduce the ``EnumWindows`` function from the user32.dll 
  library. 
* Lines #13 - 16 implement the delegate. For this simple example, we just want 
  to output the handle to the console.
* Finally, in line #19 we invoke the external method and pass in the delegate.

Both of the above examples depend on parameters, and in both cases, the parameters 
are given as managed types. Runtime does the "right thing" and processes these 
into its equivalents on the other side. Since this process is really important 
to writing quality native interop code, let's take a look at what happens when 
the runtime *marshals* the types. 

Type marshalling
^^^^^^^^^^^^^^^^
**Marshalling** is the process of transforming types when they need to cross the 
managed boundary into native and vice versa. 

The reason marshalling is needed is because the types in the managed and unmanaged world 
are different. In managed world, for instance, you have a ``String``, while in 
the unmanaged world strings can be Unicode ("wide"), non-Unicode, null-terminated, 
ASCII, etc. By default, the P/Invoke subsystem will try to do the Right Thing 
based on the default behavior which you can see on `MSDN <https://msdn.microsoft.com/en-us/library/zah6xy75.aspx>`_.

However, for those situations where you need extra control, you can employ the 
``MarshalAs`` attribute to tell the runtime what is the expected type in the 
unmanaged world. For instance, if we want the string to be sent as a null-terminated 
ANSI string, we could do it like this:

.. code-block:: c#
    
    [DllImport("somenativelibrary.dll"]
    static extern int MethodA([MarshalAs(UnmanagedType.LPStr) string parameter);

Another aspect of type marshalling is how to pass in a struct to an unmanaged method.
For instance, some of the unmanaged methods require a struct as a parameter. 
In these cases, we need to create a corresponding struct or a class in managed 
part of the world to use it as a parameter. However, just defining the class is 
not enough, we also need to instruct the marshaler how to map fields in the class 
to the unmanaged struct. This is where the ``StructLayout`` attribute comes into 
play. 

.. code-block:: c#
    :linenos:

    [DllImport("kernel32.dll")]
    static extern void GetSystemTime(SystemTime systemTime);

    [StructLayout(LayoutKind.Sequential)]
    class SystemTime {
        public ushort Year;
        public ushort Month;
        public ushort DayOfWeek;
        public ushort Day;
        public ushort Hour;
        public ushort Minute;
        public ushort Second;
        public ushort Milsecond;
    }

    public static void Main(string[] args) {
        SystemTime st = new SystemTime();
        GetSystemTime(st);
        Console.WriteLine(st.Year);
    }

The example above shows off a simple example of calling into ``GetSystemTime()`` 
function. The interesting bit is on line 4. The attribute specifies that the 
fields of the class should be mapped sequentially on pack-sized boundaries, 
similarly to the way a C struct is packed. It also means that the field names 
in the class are not important; only their order is important, and it needs to
correspond to its unmanaged target, which is shown below:

.. code-block:: c

    typedef struct _SYSTEMTIME {
      WORD wYear;
      WORD wMonth;
      WORD wDayOfWeek;
      WORD wDay;
      WORD wHour;
      WORD wMinute;
      WORD wSecond;
      WORD wMilliseconds;
    } SYSTEMTIME, *PSYSTEMTIME;
    

COM interoperability
--------------------
COM stands for **Component Object Model**. The idea behind COM was to facilitate code 
reuse by allowing libraries to define the contract of the functionality they 
provide separate from the implementation. These contracts, or *interfaces* in 
COM terminology, are the primary types that you deal with. They are similar in 
some regard to C# interfaces (or Java interfaces), but have some peculiarities; 
the entire scope of writing COM objects is beyond the scope of this article, however 
there are some resources in the `More resources`_ section. 

However, it is good to note two very important things about COM interop: 
1. COM interop is available only on Windows.
2. It is available on the desktop .NET Framework and not available on .NET Core. 
You can read more about various editions of .NET in the :doc:`../getting-started/overview` topic.

Interoperability between COM objects and managed code is similar to the way 
P/Invoke works. In the managed world, you don't deal with COM types, you deal 
with C# objects, and the runtime marshals your calls into those objects to the 
COM subsystem using something that is called **Runtime-Callable Wrappers (RCW)**. 
Runtime also does all of the house cleaning, such as object life cycle, type 
conversions and similar. 

These wrappers are exposed in your code by generating proxy types for the managed 
language that you want. This is done via the **tlimp.exe** tool command-line 
tool (the full name is Type Library Importer). This tool will consume the COM 
interface that you point it to, and generate a *COM interop asembly*, which will 
contain managed types that correspond to the interfaces. You can then reference 
these this assembly from your code and work with the objects like they are managed 
types. 

As with P/Invoke, COM interoperability allows managed types to be exposed to the 
COM subsystem. This is done through a proxy called **COM-Callable Wrappers (CCW)**. 
They operate in the same manner as RCW, only in different direction, from COM into 
managed world. They also implement the basic required interfaces by the COM 
protocol, ``IUnknown`` and ``IDispatch``. The way to expose managed types is to 
first define an assembly attribute that specifies a GUID; this GUID identifies 
the COM type library. We then use the **tlbexp.exe** (Type Library Exporter) 
command line tool to generate a COM type library. By default, all public members 
of the managed type are visible to the consuming COM code. You can control this 
using the ``ComVisible`` attribute on specific members of the type. 

Of course, this is just scratching the surface of COM interoperability, and if 
you dig into this topic, you will soon find more details. Also, 

More resources
--------------

* `PInvoke.net wiki <http://www.pinvoke.net>`_ an excellent Wiki with information 
  on common Win32 APIs and how to call them.
* `P/Invoke on MSDN <https://msdn.microsoft.com/en-us/library/zbz07712.aspx>`_
* `Mono documentation on P/Invoke <http://www.mono-project.com/docs/advanced/pinvoke/>`_ 
* `COM basics <https://msdn.microsoft.com/en-us/library/windows/desktop/ms694363(v=vs.85).aspx>`_ 
* `COM Interop on MSDN <https://msdn.microsoft.com/en-us/library/z6tx9dw3.aspx>`_
* `tlimp.exe reference <https://msdn.microsoft.com/en-us/library/tt0cf3sx%28v=vs.110%29.aspx?f=255&MSPPError=-2147217396>`_
* `tlbexp.exe reference <https://msdn.microsoft.com/en-us/library/hfzzah2c(v=vs.110).aspx>`_ 
