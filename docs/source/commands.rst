.. _commands:

.. |argparse| replace:: ``argparse``
.. _argparse: http://docs.python.org/dev/library/argparse.html

Command Line Pecan
==================
Any Pecan application can be controlled and inspected from the command line
using the built-in ``pecan`` command.  The usage examples of the ``pecan``
command in this document are intended to be invoked from your project's root
directory.  

Serving a Pecan App For Development
-----------------------------------
Pecan comes bundled with a lightweight WSGI development server based on
Python's ``wsgiref.simpleserver`` module.

Serving your Pecan app is as simple as invoking the ``pecan serve`` command::

    $ pecan serve config.py
    Starting server in PID 000.
    serving on 0.0.0.0:8080, view at http://127.0.0.1:8080

...and then visiting it in your browser.

The server ``host`` and ``port`` in your configuration file can be changed as
described in :ref:`server_configuration`.

.. include:: reload.rst
    :start-after: #reload

The Interactive Shell
---------------------
Pecan applications also come with an interactive Python shell which can be used
to execute expressions in an environment very similar to the one your
application runs in.  To invoke an interactive shell, use the ``pecan shell``
command::

    $ pecan shell config.py
    Pecan Interactive Shell
    Python 2.7.1 (r271:86832, Jul 31 2011, 19:30:53)
    [GCC 4.2.1 (Based on Apple Inc. build 5658)
    
      The following objects are available:
      wsgiapp    - This project's WSGI App instance
      conf       - The current configuration
      app        - webtest.TestApp wrapped around wsgiapp

    >>> conf
    Config({
        'app': Config({
            'root': 'myapp.controllers.root.RootController',
            'modules': ['myapp'],
            'static_root': '/Users/somebody/myapp/public', 
            'template_path': '/Users/somebody/myapp/project/templates',
            'errors': {'404': '/error/404'},
            'debug': True
        }),
        'server': Config({
            'host': '0.0.0.0',
            'port': '8080'
        })
    })
    >>> app
    <webtest.app.TestApp object at 0x101a830>
    >>> app.get('/')
    <200 OK text/html body='<html>\n ...\n\n'/936>

Press ``Ctrl-D`` to exit the interactive shell (or ``Ctrl-Z`` on Windows).

Using an Alternative Shell
++++++++++++++++++++++++++
``pecan shell`` has optional support for the `IPython <http://ipython.org/>`_
and `bpython <http://bpython-interpreter.org/>`_ alternative shells, each of
which can be specified with the ``--shell`` flag (or its abbreviated alias,
``-s``), e.g.,
::

    $ pecan shell --shell=ipython config.py
    $ pecan shell -s bpython config.py

Extending ``pecan`` with Custom Commands
----------------------------------------
While the commands packaged with Pecan are useful, the real utility of its
command line toolset lies in its extensibility.  It's convenient to be able to
write a Python script that can work "in a Pecan environment" with access to
things like your application's parsed configuration file or a simulated
instance of your application itself (like the one provided in the ``pecan
shell`` command).

Writing a Custom Pecan Command
++++++++++++++++++++++++++++++

As an example, let's create a command that can be used to issue a simulated
HTTP GET to your application and print the result.  Its invocation from the
command line might look something like this::

    $ pecan wget config.py /path/to/some/resource

Let's say you have a distribution with a package in it named ``myapp``, and
that within this package is a ``wget.py`` module::

    # myapp/myapp/wget.py
    import pecan
    from webtest import TestApp

    class GetCommand(pecan.commands.BaseCommand):
        '''
        Issues a (simulated) HTTP GET and returns the request body.
        '''

        arguments = pecan.commands.BaseCommand.arguments + ({
            'name': 'path',
            'help': 'the URI path of the resource to request'
        },)

        def run(self, args):
            super(GetCommand, self).run(args)
            app = TestApp(self.load_app())
            print app.get(args.path).body

Let's analyze this piece-by-piece.

Overriding the ``run`` Method
,,,,,,,,,,,,,,,,,,,,,,,,,,,,,

First, we're subclassing ``pecan.commands.BaseCommand`` and extending
the ``run`` method to: 

* Load a Pecan application - ``self.load_app()``
* Wrap it in a fake WGSI environment - ``webtest.TestApp()``
* Issue an HTTP GET request against it - ``app.get(args.path)``

Defining Custom Arguments
,,,,,,,,,,,,,,,,,,,,,,,,,

The ``arguments`` class attribute is used to define command line arguments
specific to your custom command.  You'll notice in this example that we're
*adding* to the arguments list provided by ``pecan.commands.BaseCommand``
(which already provides an argument for the ``config_file``), rather
than overriding it entirely.

The format of the ``arguments`` class attribute is a *tuple* of dictionaries,
with each dictionary representing an argument definition in the
same format accepted by Python's |argparse|_ module (more specifically,
``argparse.ArgumentParser.add_argument``).  By providing a list of arguments in
this format, the ``pecan`` command can include your custom commands in the help
and usage output it provides::

    $ pecan -h
    usage: pecan [-h] command ...

    positional arguments:
      command
        wget        Issues a (simulated) HTTP GET and returns the request body
        serve       Open an interactive shell with the Pecan app loaded
        ...

::

    $ pecan wget -h
    usage: pecan wget [-h] config_file path
    $ pecan wget config.py /path/to/some/resource

Additionally, you'll notice that the first line of ``GetCommand``'s docstring
- ``Issues a (simulated) HTTP GET and returns the request body`` - is
automatically used to describe the ``wget`` command in the output for ``$ pecan
-h``.  Following this convention allows you to easily integrate a summary for
your command into the Pecan command line tool.

Registering a Custom Command
++++++++++++++++++++++++++++
Now that you've written your custom command, you’ll need to tell your
distribution’s ``setup.py`` about its existence and reinstall.  Within your
distribution’s ``setup.py`` file, you'll find a call to ``setuptools.setup()``,
e.g., ::

    # myapp/setup.py
    ...
    setup(
        name='myapp',
        version='0.1',
        author='Joe Somebody',
        ...
    )

Assuming it doesn't exist already, we'll add the ``entry_points`` argument 
to the ``setup()`` call, and define a ``[pecan.command]`` definition for your custom
command::

    
    # myapp/setup.py
    ...
    setup(
        name='myapp',
        version='0.1',
        author='Joe Somebody',
        ...
        entry_points="""
        [pecan.command]
        wget = myapp.wget:GetCommand
        """
    )

Once you've done this, reinstall your project in development to register the
new entry point::

    $ python setup.py develop

...and give it a try::

    $ pecan wget config.py /path/to/some/resource
