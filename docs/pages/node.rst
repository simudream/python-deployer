.. _node-object:

The node object
===============

The :class:`Node <deployer.node.base.Node>` class is probably the most
important class of this framework. See :ref:`architecture of roles and nodes
<architecture-of-roles-and-nodes>` for a high level overview of what a Node
exactly is.

A simple example of a node:

::

    from deployer.node import ParallelNode

    class SayHello(ParallelNode):
        def hello(self):
            self.host.run('echo hello world')

.. note:: It is interesting to know that ``self`` is actually not a
      :class:`~deployer.node.base.Node` instance like you would expect, but an
      :class:`~deployer.node.base.Env` object which will proxy this actual Node
      class. This is because there is some metaclass magic going on, which
      takes care of sandboxing, logging and some other nice stuff, that you get
      for free.

      Except that a few other variables like :func:`self.console
      <deployer.node.base.Env.console>` are available, you normally won't notice
      anything.


Running the code
----------------

In order to run methods of a node, it has to be wrapped in an
:class:`~deployer.node.base.Env` object. This will manage execution, optional
sandboxing, logging and much more. It will also make sure that
:attr:`self.hosts <deployer.node.base.Env.hosts>` actually becomes a
:class:`~deployer.host_container.HostsContainer`, a proxy through which you can
run methods on a series of hosts.

The easiest way to wrap a node inside an :class:`~deployer.node.base.Env` is by
using the :func:`~deployer.node.base.Env.default_from_node` helper. This will
make sure that you can see the output and you can interact.

::

    from deployer.node import Env

    env = Env.default_from_node(MyNode())
    env.hello()


.. _node-inheritance:

Inheritance
-----------

A node is meant to be reusable. It is encouraged to inherit from such a node
class and overwrite properties or class members.


.. _double-underscore-expansion:

Expansion of double underscores
*******************************

The double underscore expansion is a kind of syntactic sugar to make overriding
more readable.

Suppose we already had a node like this:

::

    class WebApp(Node):
        class Nginx(Node):
            class Server(Node):
                domain = 'www.example.com'

Now, we'd like to inherit from ``WebApp``, but change the
``Nginx.Server.domain`` property there to 'mydomain.com'. Normally, in Python,
you do this:

::

    class MyWebApp(WebApp):
        class Nginx(WebApp.Nginx):
            class Server(WebApp.Nginx.Server):
                domain = 'mydomain.com'

This is not too bad, but if you have a lot of nested classes, it can become
pretty ugly. Therefor, the :class:`Node <deployer.node.base.Node>` class has
some magic which allows us to do this instead:

::

    class MyWebApp(WebApp):
        Nginx__Server__domain = 'mydomain.com'

If you'd like, you can also use the same syntax to add function to the inner
classes:

::

    class MyWebApp(WebApp):
        def Nginx__Server__get_full_domain(self):
            # Note that 'self' points to the 'Server' class at this point,
            # not to 'Webapp'!
            return 'http://%s' % self.domain


The importance of ``ParallelNode``
----------------------------------

There are several kind of setups. You can have many hosts which are all doing
exactly the same, or many hosts that do something different. Simply said,
:class:`ParallelNode <deployer.node.base.ParallelNode>` should be used when you
have many hosts in your node that all do exactly the same. Actions on such a
:class:`ParallelNode <deployer.node.base.ParallelNode>` can be executed in
parallel. The hosts are equal but also independend and don't need to know about
each other. An example is an array of stateless web servers.

A typical setup consists of a root node which is just a normal
:class:`Node <deployer.node.base.Node>`, with several arrays of
:class:`ParallelNode <deployer.node.base.ParallelNode>` nested inside.


Isolation of hosts in ``ParallelNode``.
***************************************

Take the following example:

::

    class WebSystem(ParallelNode):
        class Hosts:
            host = { Host1, Host2, Host3, Host4 }

        def checkout_git(self, commit):
            self.host.run("git checkout '%s'" % esc1(commit))

        def restart(self):
            self.host.run("nginx restart")

        def deploy(self, commit):
            self.checkout_git(commit)
            self.restart()


We see a :class:`ParallelNode <deployer.node.base.ParallelNode>` class with
three actions and four Hosts mapped to the role ``host`` of this node. Because
of the isolation that :class:`ParallelNode <deployer.node.base.ParallelNode>`
provides, it is possible to call any of the four actions independently on any
of the four hosts. Look how our ``WebSystem`` acts like an array:

::

    websystem = Env.default_from_node(WebSystem())
    websystem[Host1].deploy('abcde6565eee...')
    websystem[Host2].restart()

We can also call an action directly without specifying a host. This will allow
parallel execution. It says: call this action on every cell of the array. They
are independent and unordered in this case, so we don't have to run the deploy
sequentially.

::

    websystem = Env.default_from_node(WebSystem())
    websystem.deploy('abcde6565eee...') # Parallel execution.

.. note:: One thing worth noting is that there is a variable
          :attr:`~deployer.node.base.ParallelNode.host` in the class. This is
          because the isolation always happens by convention on the role named
          ``host``. Both sides of the following equation will represent a
          :class:`~deployer.host_container.HostContainer` containing exactly
          one host: the host of the current isolation.

          ::

                self.host == self.hosts.filter('host')

          If there happen to be hosts mapped to other roles, they will simply
          become available for every instance in the role named ``host``. If
          you'd call ``self.hosts.filter('other_role')``, that would still
          work.


.Array and .JustOne
*******************

``.Array`` and ``.JustOne`` are required for nesting a
:class:`ParallelNode <deployer.node.base.ParallelNode>` inside a normal
:class:`Node <deployer.node.base.Node>`. The idea is that when host roles are
mapped from the parent :class:`Node <deployer.node.base.Node>`, to the child --
which is a :class:`ParallelNode <deployer.node.base.ParallelNode>` --, that
this childnode behaves as an array. Each 'cell' in the array is isolated, so
it's possible to execute a command on just one 'cell' (or host) of the array or
all 'cells' (or hosts.) You can use it as follows:

::

    class NormalNode(Node):
        class OurParallelNode(ParallelNode.Array):
            class PNode(ParallelNode):
                pass


Basically, you can nest 'normal' nodes inside each other, and
:class:`ParallelNode <deployer.node.base.ParallelNode>` classes inside each
other. However, when nesting such a :class:`ParallelNode
<deployer.node.base.ParallelNode>` inside a normal node, the ``.Array`` suffix
is required to indicate the creation of an array. ``.JustOne`` can always be
used instead of an array, if you assert that only one host will be in there.


Using contrib.nodes
-------------------

The deployer framework is delivered with a `contrib.nodes` directory which
contains nodes that should be generic enough to be usable by a lot of people.
Even if you can't use them in your case, they may be good examples of how to do
certain things. So don't be afraid to look at the source code, you can learn some
good practices there. Take these and inherit as you want to, or start from
scratch if you prefer that way.

Some recommended contrib nodes:

 - `deployer.contrib.nodes.config.Config`

   This a the base class that we are using for every configuration file. It is
   very useful for when you are automatically generating server configurations
   according to specific deployment configurations. Without any efford, this
   class will allow you to do diff's between your new, generated config, and
   the config that's currently on the server side.


Reference
---------

See :ref:`Node reference <node-reference>`.
