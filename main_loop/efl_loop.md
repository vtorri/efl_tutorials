~~Getting started with the EFL - the main loop~~

The EFL is a set of libraries which helps you writing application,
from a simple terminal one to a powerful GUI. This tutorial will cover
how to write a text application and how to deal with the main loop,
using thee new unified API.

==== Basics ====

We will start with simple application which will display the version
of the installed EFL in the terminal. W will improve it each time we
introduce a new feature.

Create a new file, named for example **efl_version.c** with the
following content:

<code c efl_version.c>
#include <stdio.h>

#include <Efl_Core.h>

static EAPI_MAIN void
efl_main(void *data EINA_UNUSED, const Efl_Event *ev)
{
   const Efl_Version *ver;
   Eo *main_loop;

   main_loop = ev->object;
   ver = efl_app_efl_version_get(main_loop);
   if (!ver)
   {
      printf("Can not retrieve the version of the used EFL\n");
      return;
   }

   printf("EFL version: %d.%d.%d\n", ver->major, ver->minor, ver->micro);

   efl_loop_quit(main_loop, eina_value_int_init(99));
}
EFL_MAIN();
</code>

You can compile the program above with GCC using the following
command:

  gcc -o efl_version efl_version.c `pkg-config --cflags --libs ecore`

and run the application with:

  ./efl_version

to see the version of the EFL in the terminal.

All the EFL application will have the function ''efl_main()''. It will
contain the code that will be run, a bit like the classic ''main()''
function. Just after must be the ''EFL_MAIN()'' macro which takes care
of initializing the subsystem (here, Ecore), run the main loop, and
call ''efl_main()''.

The first argument of ''efl_main()'' will never be used when
''EFL_MAIN()'' is called (it is actually always set to NULL). This is
why it is followed by ''EINA_UNUSED'' which specifies to the compiler
that this argument is not used in ''efl_main()''.

The second argument is a pointer to the structure ''Efl_Event'', which
is defined like that:

    typedef struct _Efl_Event
    {
       Efl_Object *object;
       const Efl_Event_Description *desc;
       void *info;
    } Efl_Event;

The EFL are event-driven and each of them will be specific to the
object which sends them. Here is a short description of the ''event''
members passed to ''efl_main()'':

* the ''object'' member is set to an Eo * object which represents the
  [application main loop](/develop/guides/c/core/main-loop.md). It can
  be used as the parent for any object you create that requires a main
  loop, like :

  * Timers
  * File Descriptor Monitors
  * Idlers
  * Pollers
  * Promises and Futures

* the ''desc'' member TODO to be done

* the ''info'' member is used to retrive the command line arguments
  passed to your program. It must be casted to an Efl_Loop_Arguments *
  pointer:

    typedef struct _Efl_Loop_Arguments
    {
       const Eina_Array *argv;
       Eina_Bool initialization;
    } Efl_Loop_Arguments;

  * ''argv'' is an Eina_Array containing the command line arguments.
  * ''initialization is not useful for us.

Finally, we can quit the main loop and exit cleanly the program by
calling ''efl_loop_quit()''. The first argument is the loop object we
want to quit (here, ''main_loop''), and the second argument is the
exit status code (an integer number).

=== Command Line Arguments ===

Here is an example which lists the command line arguments:

<code c efl_args_1.c>
#include <stdio.h>

#include <Efl_Core.h>

static EAPI_MAIN void
efl_main(void *data EINA_UNUSED, const Efl_Event *ev)
{
   Eo *main_loop;
   const Efl_Loop_Arguments *args;

   /* used to iterate over an Eina array: */
   char *argv;
   Eina_Array_Iterator iter;
   unsigned int i;

   main_loop = ev->object;
   args = ev->info;

   EINA_ARRAY_ITER_NEXT(args->argv, i, argv, iter)
     printf("%s\n", argv);

   efl_loop_quit(main_loop, eina_value_int_init(99));
}
EFL_MAIN();
</code>

So we just iterate over all the argument with the macro
''EINA_ARRAY_ITER_NEXT''.

Here is now a source code which parse the arguments according to the
usage of a program. The usage function is at the top of the code, and
the parsing of options is in ''efl_main''.

<code c efl_args_2.c>
#include <stdlib.h> /* atoi */
#include <string.h> /* str(n)cmp */
#include <stdio.h>

#include <Efl_Core.h>

static void
usage(const char *cmd)
{
   printf("Usage: %s [OPTION...] file\n\n", cmd);
   printf("Description of the tool\n\n");
   printf("Optional arguments:\n");
   printf("  -h, --help    show this help message and exit\n");
   printf("  -v, --version show the EFL version and exit\n");
   printf("  --verbose     verbose mode\n");
   printf("  -j, --jobs=   number of jobs \n");
}

static EAPI_MAIN void
efl_main(void *data EINA_UNUSED, const Efl_Event *ev)
{
   Eo *main_loop;
   const Efl_Loop_Arguments *args;

   /* used to iterate over an Eina array: */
   const char *argv;
   Eina_Array_Iterator iter;
   unsigned int i;

  /* values set by arguments */
  char *exe;
  Eina_Bool verbose = EINA_FALSE;
  int jobs = 1;
  const char *file = NULL;

   main_loop = ev->object;
   args = ev->info;

   /* first argument is binary, we skip it */
   iter = args->argv->data;
   exe = *(iter++);
   for (i = 1; i < eina_array_count(args->argv); i++)
   {
      argv = *(iter++);

      /* we parse options */
      if (*argv == '-')
      {
         /* we first check that all options are passed before the file */
         if (file)
         {
            usage(exe);
            efl_loop_quit(main_loop, eina_value_int_init(0));
         }

         argv++;

         if ((strcmp(argv, "h") == 0) || (strcmp(argv, "-help") == 0))
         {
            usage(exe);
            efl_loop_quit(main_loop, eina_value_int_init(0));
         }
         if ((strcmp(argv, "v") == 0) || (strcmp(argv, "-version") == 0))
         {
            const Efl_Version *ver;
            ver = efl_app_efl_version_get(main_loop);
            printf("EFL %d.%d.%d\n", ver->major, ver->minor, ver->micro);
            efl_loop_quit(main_loop, eina_value_int_init(0));
         }
         else if (strcmp(argv, "-verbose") == 0)
         {
            verbose = EINA_TRUE;
         }
         if ((strcmp(argv, "j") == 0) ||
             (strncmp(argv, "-jobs=", strlen("-jobs=")) == 0))
         {
            const char *j_s;

            if (strcmp(argv, "j") == 0)
            {
               if (i < (eina_array_count(args->argv) - 1))
               {
                  i++;
                  j_s = *(iter++);
               }
               else
               {
                  usage(exe);
                  efl_loop_quit(main_loop, eina_value_int_init(1));
               }
            }
            else
            {
               j_s = argv + 6; /* strlen("-jobs=") equals 6 */
            }

            /* should use strtod for errors and check it's >= 1 */
            jobs = atoi(j_s);
         }
      }
      else
      {
         if (!file)
            file = argv;
         else
         {
            /* if another argument is passed after the file, we exit */
            usage(exe);
            efl_loop_quit(main_loop, eina_value_int_init(1));
         }
      }
   }

   printf(" * verbose : %s\n", verbose ? "yes" : "no");
   printf(" * jobs    : %d\n", jobs);
   printf(" * file    : %s\n", file);

   efl_loop_quit(main_loop, eina_value_int_init(99));
}
EFL_MAIN();
</code>

=== Timers ===

Timers can be added to the main loop Eo object using the ''efl_add''
macro. A callback is then called when an interval of time has elapsed:

    efl_add(EFL_LOOP_TIMER_CLASS, main_loop,
            efl_loop_timer_interval_set(efl_added, 2.0),
            efl_event_callback_add(efl_added, EFL_LOOP_TIMER_EVENT_TIMER_TICK, my_cb, data));

Some explanations:

* timers are Eo objects and ''efl_add'' created it and sets it as the
child of main_loop. The destruction is done automatically when exiting
the main loop. Each Eo object has a class. Timer's class is
''EFL_LOOP_TIMER_CLASS''.
* ''efl_loop_timer_interval_set'' method sets the interval of time at
the end of which the callback will be called. The special symbol
''efl_added'' refers to the object being created. It's the analog of
the **this** pointer in C++ classes.
* ''efl_event_callback_add'' add a callback for this timer, ''my_cb''
is the callback name, the last parameter is the data passed to that
callback.

The declaration of the callbck is the following:

    void my_cb(void *data, const Efl_Event *ev);

Here is a basic example of a timer calling a callback, and which exits
the program at the third call of the callback:

<code c efl_timer_1.c>
#include <stdio.h>

#include <Efl_Core.h>

static double initial_time = 0;

static void
_cb(void *data, const Efl_Event *ev EINA_UNUSED)
{
   Eo *main_loop = (Eo*)data;

   double dt = ecore_time_get() - initial_time;
   printf("%.6f\n", dt);
   if (dt > 5)
     efl_loop_quit(main_loop, eina_value_int_init(99));
}

static EAPI_MAIN void
efl_main(void * data EINA_UNUSED, const Efl_Event *ev)
{
   Eo *main_loop;

   main_loop = ev->object;
   initial_time = ecore_time_get();

   efl_add(EFL_LOOP_TIMER_CLASS, efl_app_main_get(),
           efl_loop_timer_interval_set(efl_added, 2.0),
           efl_event_callback_add(efl_added, EFL_LOOP_TIMER_EVENT_TIMER_TICK,
                                  _cb, main_loop));
}
EFL_MAIN();
</code>

Here is a rewrite of the file examples/ecore/ecore_timer_example.c
with the unifies API:

<code c efl_timer_2.c>
#include <stdio.h>

#include <Efl_Core.h>

#define TIMEOUT_1 1.0  // interval for timer1
#define TIMEOUT_2 3.0  // timer2 - delay timer1
#define TIMEOUT_3 8.2  // timer3 - pause timer1
#define TIMEOUT_4 11.0 // timer4 - resume timer1
#define TIMEOUT_5 14.0 // timer5 - change interval of timer1
#define TIMEOUT_6 18.0 // top timer1 and start timer7 and timer8 with changed precision
#define TIMEOUT_7 1.1  // interval for timer7
#define TIMEOUT_8 1.2  // interval for timer8
#define DELAY_1   3.0  // delay time for timer1 - used by timer2
#define INTERVAL1 2.0  // new interval for timer1 - used by timer5

#define TIMER_ADD(timeout_, cb_) \
   efl_add(EFL_LOOP_TIMER_CLASS, efl_app_main_get(), \
           efl_loop_timer_interval_set(efl_added, timeout_), \
           efl_event_callback_add(efl_added, EFL_LOOP_TIMER_EVENT_TIMER_TICK, \
                                  cb_, NULL))

static double _initial_time = 0;

static Eo *timer1 = NULL;
static Eo *timer2 = NULL;
static Eo *timer3 = NULL;
static Eo *timer4 = NULL;
static Eo *timer5 = NULL;
static Eo *timer6 = NULL;
static Eo *timer7 = NULL;
static Eo *timer8 = NULL;

static inline double
_get_current_time(void)
{
   return ecore_time_get() - _initial_time;
}

static void
_timer1_cb(void *data EINA_UNUSED, const Efl_Event *ev EINA_UNUSED)
{
   printf("Timer1 expired after %0.3f seconds.\n", _get_current_time());
   fflush(stdout);
}

static void
_timer2_cb(void *data EINA_UNUSED, const Efl_Event *ev EINA_UNUSED)
{
   printf("Timer2 expired after %0.3f seconds. "
          "Adding delay of %0.3f seconds to timer1.\n",
          _get_current_time(), DELAY_1);
   fflush(stdout);

   efl_loop_timer_delay(timer1, DELAY_1);

   efl_event_callback_stop(timer2);
   efl_del(timer2);
   timer2 = NULL;
}

static void
_timer3_cb(void *data EINA_UNUSED, const Efl_Event *ev EINA_UNUSED)
{
   printf("Timer3 expired after %0.3f seconds. "
          "Freezing timer1.\n", _get_current_time());
   fflush(stdout);

   efl_event_freeze(timer1);

   efl_event_callback_stop(timer3);
   efl_del(timer3);
   timer3 = NULL;
}

static void
_timer4_cb(void *data EINA_UNUSED, const Efl_Event *ev EINA_UNUSED)
{
   printf("Timer4 expired after %0.3f seconds. "
          "Resuming timer1, which has %0.3f seconds left to expire.\n",
          _get_current_time(), efl_loop_timer_time_pending_get(timer1));
   fflush(stdout);

   efl_event_thaw(timer1);

   efl_event_callback_stop(timer4);
   efl_del(timer4);
   timer4 = NULL;
}

static void
_timer5_cb(void *data EINA_UNUSED, const Efl_Event *ev EINA_UNUSED)
{
   double interval = efl_loop_timer_interval_get(timer1);

   printf("Timer5 expired after %0.3f seconds. "
          "Changing interval of timer1 from %0.3f to %0.3f seconds.\n",
          _get_current_time(), interval, INTERVAL1);
   fflush(stdout);

   efl_loop_timer_interval_set(timer1, INTERVAL1);

   efl_event_callback_stop(timer5);
   efl_del(timer5);
   timer5 = NULL;
}

static void
_timer7_cb(void *data EINA_UNUSED, const Efl_Event *ev EINA_UNUSED)
{
   printf("Timer7 expired after %0.3f seconds.\n", _get_current_time());
   fflush(stdout);

   efl_event_callback_stop(timer7);
   efl_del(timer7);
   timer7 = NULL;
}

static void
_timer8_cb(void *data EINA_UNUSED, const Efl_Event *ev EINA_UNUSED)
{
   printf("Timer8 expired after %0.3f seconds.\n", _get_current_time());
   fflush(stdout);

   efl_event_callback_stop(timer8);
   efl_del(timer8);
   timer8 = NULL;
}

static void
_timer6_cb(void *data EINA_UNUSED, const Efl_Event *ev EINA_UNUSED)
{
   printf("Timer6 expired after %0.3f seconds.\n", _get_current_time());
   printf("Stopping timer1.\n");
   fflush(stdout);

   efl_del(timer1);
   timer1 = NULL;

   printf("Starting timer7 (%0.3fs) and timer8 (%0.3fs).\n",
          TIMEOUT_7, TIMEOUT_8);
   fflush(stdout);

   timer7 = TIMER_ADD(TIMEOUT_7, _timer7_cb);
   timer8 = TIMER_ADD(TIMEOUT_8, _timer8_cb);

   /* FIXME */
   // ecore_timer_precision_set(0.2);

   efl_event_callback_stop(timer6);
   efl_del(timer6);
   timer6 = NULL;
}

static EAPI_MAIN void
efl_main(void * data EINA_UNUSED, const Efl_Event *ev)
{

   _initial_time = ecore_time_get();

   timer1 = TIMER_ADD(TIMEOUT_1, _timer1_cb);
   timer2 = TIMER_ADD(TIMEOUT_2, _timer2_cb);
   timer3 = TIMER_ADD(TIMEOUT_3, _timer3_cb);
   timer4 = TIMER_ADD(TIMEOUT_4, _timer4_cb);
   timer5 = TIMER_ADD(TIMEOUT_5, _timer5_cb);
   timer6 = TIMER_ADD(TIMEOUT_6, _timer6_cb);

   printf("start the main loop.\n");
   fflush(stdout);
}
EFL_MAIN();
</code>
