# Testing Stateful Server Networks with ScalaCheck

This example project demonstrates how ScalaCheck's `Commands`
[API](http://scalacheck.org/files/scalacheck_2.11-1.12.2-api/index.html#org.scalacheck.commands.Commands)
can be combined with [NixOS](http://nixos.org) in order to test entire server
networks in an automated, property-based fashion.

This is a proof-of-concept project, and many corners have been cut. The
integration between ScalaCheck and Nix/NixOS is one example of something that
deserves it own dedicated project in the future.

The test code also has a very naive way of communicating with the test machines
through SSH, and a hideous method of using `sleep` to wait for machines to
boot. These are clearly things not suitable for production.

## What is tested?

On each property evaluation, ScalaCheck generates a network of (initially
stopped) servers. In the example code, the network consists of between 1 and 4
nodes. Then, a sequence of commands is generated and executed. A command is
either `Boot`, for booting up a server, or `Ping(from, to)` for sending a ICMP
PING from one machine to another. The postcondition of the `Ping` command is
simple: if an online machine is pinged, it is expected to reply. If an offline
machine is pinged, it is expected not to reply.

This is of course a very simple and stupid test, but it demonstrates what can
be done. Each node in the test network could just as well have ran a
distributed database server. The commands could have verified the semantics of
the database while adding, removing and restarting different nodes.

Also, the use of NixOS turns all this into true integration testing. *Any*
component of the complete system can be parameterised and handled by
ScalaCheck's ordinary generators and test case simplification machinery. For
example, do you suspect that the file system your database server is writing
its state to can impact the semantics? Then simply make that one parameter in
your model and let ScalaCheck try out any combination of `ext4`, `xfs`, `btrfs`
etc on your test machines. The same goes for things like versions of different
libraries, programs, services, kernels etc that your system makes use of. You
can also do regression testing by for example testing a new server
implementation against a range of old clients.

The example project demonstrates simple use of parameters by modeling the
memory size and the Linux kernel version of the test machines.


## Prerequisites

To run the tests you need to have a working [Nix](http://nixos.org/nix) setup.
Nix runs on any Linux distribution and OSX, but I haven't tested this
particular project on any other OS than NixOS.

You also need to have [libvirt](http://libvirt.org) working together with
[QEMU](http://qemu.org) (`qemu:///session`).

Finally, you need to setup the [VDE](http://vde.sourceforge.net) virtual
ethernet switch to handle QEMU networking. The code in this project assumes you
have a VDE switch running at `/run/vde0.ctl` that is writeable by your local
user. It is also assumed that your host machine has a VDE interface with an IP
number in the `172.16.2.0/24` subnet. You can of course make relevant changes
in the Scala and Nix code to adjust for your own environment.

You should first and foremost look at this project as proof-of-concept. It is
not a neatly packaged solution, but is meant to act as a demonstration and
source of inspiration.

Also note that there is nothing inherent to QEMU and VDE in this project. They
could be replaced by other similar components. The core parts are ScalaCheck
Commands API and Nix/NixOS. You could probably even replace Nix/NixOS too, by
generating virtual machines (or containers) by some other means. The strength
of NixOS is that you have endless possibilities of customising your machines.
NixOS is also a good candidate for running production systems, not only tests.
If you do that, you could make your test systems very similar to the production
system.


## How does it work?

The system state model looks like this:

```scala
case class Machine (
  id: String,
  uuid: java.util.UUID,
  ip: String,
  kernelVer: String,
  memory: Int,
  running: Boolean
)

type State = List[Machine]
```

The state is just the current collection of machine states. The machine state
stores parameters like IP number, memory amount and current running status.

The system under test type (`Commands.Sut`), used for communication with the
real system looks like this:

```scala
type Sut = Map[String, org.libvirt.Domain]
```

Each `Machine.id` is mapped to an instance of `org.libvirt.Domain` which is
a handle to a [libvirt](http://libvirt.org) virtual machine.

A new `Sut` instance is created in the following way (some error handling has
been removed for clarity):

```scala
val con = new org.libvirt.Connect("qemu:///session")

def newSut(state: State): Sut = {
  toLibvirtXMLs(state) map { case (id,xml) =>
    id -> con.domainDefineXML(xml)
  }
}
```

First a connection to user's local libvirt QEMU manager is established. The
method `toLibvirtXMLs()` (more on this soon) takes a `State` instance and
creates libvirt XML-files that describes the virtual machines we want.
`con.domainDefineXML` creates a virtual machine and returns a handle to it. We
look at the state and boots up any server that should be running from start
with the `con.create()` method.

`toLibvirtXMLs` works in two steps. First, it generates a NixOS configuration
for each machine in the network. This is done by simply injecting the values
from `Machine` into a base NixOS configuration:

```scala
def toNixNetwork(machines: Iterable[Machine]): String = {
  def mkConf(m: Machine): String = raw"""
    ${m.id} = { config, pkgs, lib, ... }: {
      imports = [ ./common.nix ];
      ${toNixMachine(m)}
    };
  """
  s"import ./qemu-network.nix { ${machines.map(mkConf).mkString} }"
}

def toNixMachine(m: Machine): String = raw"""
  deployment.libvirt = {
    netdevs.netdev0.mac = "$$MAC0";
    memory = ${m.memory};
    uuid = "${m.uuid}";
  };
  networking.hostName = "${m.id}";
  networking.interfaces.eth0 = {
    ipAddress = "${m.ip}";
    prefixLength = 24;
  };
  boot.kernelPackages =
    pkgs.linuxPackages_${m.kernelVer.replace('.','_')};
"""
```

The module `qemu-network.nix` used above is also part of this example project
and is a Nix module that takes the configuration of a set of machines and then
builds all software, configuration files, kernel, initrd etc that is needed by
the QEMU machine. In the end, it creates a libvirt XML file for each machine,
that assembles all parts into a file that can be used by libvirt to create and
start the virtual machine.

By clever use of NixOS we don't have to generate any virtual disk images.
Instead the virtual machines mounts their system files directly from the host
machine. That way, all virtual machines are able to share the same files if
they don't differ. The only thing that limits the number of simultaneous
machines is then only CPU and memory resources.

The `toLibvirtXMLs()` method uses `toNixNetwork()` to create a file containing
the NixOS configurations and then calls out to `nix-build` to build the libvirt
XML files.

### Generating the state

The `State` generator looks like this:

```scala
def genMachine(id: String, subnet: List[Int]): Gen[Machine] = for {
  uuid <- Gen.uuid
  ip <- Gen.choose(2,254).map(n => s"172.16.2.$n")
  memory <- Gen.choose(96, 256)
  kernel <- Gen.oneOf("3.14", "3.13", "3.12", "3.10")
} yield Machine (id, uuid, ip, kernel, memory, false)

val genInitialState: Gen[State] = for {
  machineCount <- Gen.choose(1,4)
  idGen = Gen.listOfN(8, Gen.alphaLowerChar).map(_.mkString)
  ids <- Gen.listOfN(machineCount, idGen)
  subnet <- genSubnet
  machines <- Gen.sequence[List,Machine](ids.map(genMachine(_, subnet)))
} yield machines
```

In the example code the IP number is hardcoded to the subnet `172.16.2.0/24`
because that is how my laptop is setup. We pick a memory amount between 96 and
256 MB, and selects a Linux kernel version from a list of alternatives.

### The Commands

Only three commands are supported, `Boot`, `Shutdown` and `Ping`:

```scala
case class Boot(m: Machine) extends Command {
  type Result = Boolean
  def run(sut: Sut) = {
    println(s"booting machine ${m.ip}...")
    sut(m.id).create()
    var n = 0
    while (n < 20) {
      Thread.sleep(500)
      try {
        runSshCmd(m.ip, "true")
        println(s"machine ${m.ip} is up!")
        n = Int.MaxValue
      } catch { case e: Throwable => n = n + 1 }
    }
    sut(m.id).isActive != 0 && n == Int.MaxValue
  }
  def nextState(state: State) =
    state.filterNot(_.id == m.id) :+ m.copy(running = true)
  def preCondition(state: State) = !m.running
  def postCondition(state: State, result: Try[Boolean]) =
    result == Success(true)
}

case class Shutdown(m: Machine) extends Command {
  type Result = Boolean
  def run(sut: Sut) = {
    println(s"shutting down machine ${m.ip}...")
    sut(m.id).destroy()
    sut(m.id).isActive == 0
  }
  def nextState(state: State) =
    state.filterNot(_.id == m.id) :+ m.copy(running = false)
  def preCondition(state: State) = m.running
  def postCondition(state: State, result: Try[Boolean]) =
    result == Success(true)
}

case class Ping(from: Machine, to: Machine) extends Command {
  type Result = Boolean
  def run(sut: Sut) =
    runSshCmd(from.ip, s"fping -c 1 ${to.ip}") match {
      case Right(out) =>
        println(s"${from.ip} -> $out")
        true
      case Left(out) =>
        println(s"${from.ip} -> $out")
        false
    }
  def nextState(state: State) = state
  def preCondition(state: State) = from.running
  def postCondition(state: State, result: Try[Boolean]) =
    result == Success(to.running)
}
```

The `Boot` command simply tells libvirt to start a machine and then waits until
the machine is available over ssh. This is a bit of a waste of time, since the
boot process could be done in parallel for several machines.

The `Ping` command executes the `fping` command through SSH on one machine and
expects it to succeed if the current state of the `to` machine indicates it is
running.

The `Shutdown` command destroys a running machine (which can later be booted
again).

The commands are generated in the following way:

```scala
def genPingOffline(state: State): Gen[Ping] = for {
  from <- Gen.oneOf(state.filter(_.running))
  to <- Gen.oneOf(state.filter(!_.running))
} yield Ping(from, to)

def genPingOnline(state: State): Gen[Ping] = for {
  from <- Gen.oneOf(state.filter(_.running))
  to <- Gen.oneOf(state.filter(_.running))
} yield Ping(from, to)

def genBoot(state: State): Gen[Boot] = Gen.oneOf(
  state.filterNot(_.running).map(Boot)
)

def genShutdown(state: State): Gen[Shutdown] = Gen.oneOf(
  state.filter(_.running).map(Shutdown)
)

def genCommand(state: State): Gen[Command] =
  if(state.forall(!_.running)) genBoot(state)
  else if(state.forall(_.running)) Gen.frequency(
    (1, genShutdown(state)),
    (4, genPingOnline(state))
  )
  else Gen.frequency(
    (2, genBoot(state)),
    (1, genShutdown(state)),
    (4, genPingOnline(state)),
    (4, genPingOffline(state))
  )
```


## Running the tests

There seems to be some problem getting sbt to properly print exception stack
traces, so I recommend that you run your first test run in the following way:

```
sbt "test:runMain CommandsNix" \
  -minSuccessfulTests 1 -minSize 5 -maxSize 10 \
  -verbosity 2
```
