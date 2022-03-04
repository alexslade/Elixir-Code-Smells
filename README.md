# [Catalog of Elixir-specific code smells][Elixir Smells]

[![GitHub last commit](https://img.shields.io/github/last-commit/lucasvegi/Elixir-Code-Smells)](https://github.com/lucasvegi/Elixir-Code-Smells/commits/main)
[![Twitter URL](https://img.shields.io/twitter/url?style=social&url=https%3A%2F%2Fgithub.com%2Flucasvegi%2FElixir-Code-Smells)](https://twitter.com/intent/tweet?text=Catalog%20of%20Elixir-specific%20code%20smells:&url=https%3A%2F%2Fgithub.com%2Flucasvegi%2FElixir-Code-Smells)

## Table of Contents

* __[Introduction](#introduction)__
* __[Design-related smells](#design-related-smells)__
  * [GenServer Envy](#genserver-envy)
  * [Agent Obsession](#agent-obsession)
  * [Unsupervised process](#unsupervised-process)
  * [Large messages between processes](#large-messages-between-processes)
  * [Complex multi-clause function](#complex-multi-clause-function)
  * [Complex API error handling](#complex-api-error-handling)
  * [Exceptions for control-flow](#exceptions-for-control-flow)
  * [Untested polymorphic behavior](#untested-polymorphic-behavior)
  * [Code organization by process](#code-organization-by-process)
  * [Data manipulation by migration](#data-manipulation-by-migration)
* __[Low-level concerns smells](#low-level-concerns-smells)__
  * [Working with invalid data](#working-with-invalid-data)
  * [Map/struct dynamic access](#mapstruct-dynamic-access)
  * [Unplanned value extraction](#unplanned-value-extraction)
  * [Modules with identical names](#modules-with-identical-names)
  * [Unnecessary macro](#unnecessary-macro)
  * [App configuration for code libs](#app-configuration-for-code-libs)
  * [Compile-time app configuration](#compile-time-app-configuration)
  * [Dependency with "use" when an "import" is enough](#dependency-with-use-when-an-import-is-enough)
* __[About](#about)__

## Introduction

Elixir is a new functional programming language whose popularity is rising in the industry <sup>[link][ElixirInProduction]</sup>. However, there are few works in the scientific literature focused on studying the internal quality of systems implemented in this language.

In order to better understand what are the types of sub-optimal code structures that can harm the internal quality of Elixir systems, we scoured websites, blogs, forums, and videos (grey literature review), looking for specific code smells for Elixir that are discussed by its developers.

As a result of this investigation, we proposed a catalog of 18 new smells specific to Elixir systems. These code smells have been categorized into two different groups, according to the type of impact and code extent they affect. This catalog of Elixir-specific code smells is presented below. Each code smell is documented using the following structure:

* __Name:__ Unique identifier of the code smell. This name is important to facilitate communication between developers;
* __Category:__ The portion of code affected by smell and its severity;
* __Problem:__ How the code smell can harm code quality and what impacts this can have for developers;
* __Example:__ Codes and textual descriptions to illustrate the occurrence of the code smell;
* __Refactoring:__ Ways to change smelly codes in order to improve its qualities. Examples of refactored code are presented to illustrate these changes.

The objective of this catalog of code smells is to instigate the improvement of the quality of code developed in Elixir. For this reason, we are interested in knowing Elixir's community opinion about these code smells: *Do you agree that these code smells can be harmful? Have you seen any of them in production code? Do you have any suggestions about some Elixir-specific code smell not cataloged by us?...*

Please feel free to make pull requests and suggestions ([Discussions][Discussions] tab). We want to hear you!

## Design-related smells

Design-related smells are more complex, affect a coarse-grained code element, and are therefore harder to detect. Next, all 10 different smells classified as design-related are explained and exemplified:

### GenServer Envy

* __Category:__ Design-related smell.

* __Problem:__ In Elixir, processes can be primitively created by ``spawn/1`` and ``spawn_link/1`` functions. Although it is possible to create it this way, it is more common to use abstractions (e.g., [``Agent``][Agent], [``Task``][Task], and [``GenServer``][GenServer]) provided by Elixir to create processes. The use of each specific abstraction is not a code smell in itself, however, can be trouble when either a ``Task`` or ``Agent`` are used beyond its suggested purposes, being treated like ``GenServers``.

* __Example:__ As shown next, ``Agent`` and ``Task`` are abstractions to create processes with specialized purposes. In contrast, ``GenServer`` is a more generic abstraction used to create processes for many different purposes:

  * ``Agent``: As Elixir works on the principle of immutability, by default no value is shared between multiple places of code, abling read and write such as in a global variable. An ``Agent`` is a simple process abstraction focused on solving this limitation, abling processes to share states.
  * ``Task``: This process abstraction is used when we only need to execute asynchronously some specific action, often in an isolated way, without communication with other processes.
  * ``GenServer``: Is the most generic process abstraction. The main benefit of this abstraction is explicitly segregating the server and the client roles, thus providing a better API for the organization of processes communication. Besides that, ``GenServer`` can also encapsulate state (like an ``Agent``), provide sync and async calls (like a ``Task``), and more.
  
  Examples of this code smells is when ``Agents`` or ``Tasks`` are used for general purposes and not only to specialized ones such as its documentation suggests. To illustrate some smells occurrences, we will cite two specific situations. 1) When a ``Task`` is used not only to async execute an action, but also to frequently exchange messages with other processes; 2) When an ``Agent`` besides sharing some global value between processes, also frequently are used to execute isolated tasks that are not of interest to other processes.

* __Refactoring:__ When an ``Agent`` or ``Task`` goes beyond its suggested use cases and becomes painful, is better to refactor it to a ``GenServer``.

[▲ back to Index](#table-of-contents)
___

### Agent Obsession

* __Category:__ Design-related smell.

* __Problem:__ In Elixir, an ``Agent`` is a process abstraction focused on sharing information between processes by means of message passing. It is a simple wrapper around shared information, thus facilitating its read and update on any place of a code. The use of an ``Agent`` to share information is not a code smell in itself, however, when the responsibility for interacting directly with an ``Agent`` is spread across the entire system, this can be problematic. This bad practice can difficult the code maintenance and leave the code bug-proneness.

* __Example:__ The following codes seek to illustrate this smell. The responsibility for interacting directly with the ``Agent`` is spread across four different modules (i.e, ``A``, ``B``, ``C``, and ``D``).

  ```elixir
  defmodule A do
    #...
    def update(pid) do
      #...
      Agent.update(pid, fn _list -> 123 end)
      #...
    end
  end
  ```

  ```elixir
  defmodule B do
    #...
    def update(pid) do
      #...
      Agent.update(pid, fn content -> %{a: content} end)
      #...
    end
  end
  ```

  ```elixir
  defmodule C do
    #...
    def update(pid) do
      #...
      Agent.update(pid, fn content -> [:atom_value | [content]] end)
      #...
    end
  end
  ```

  ```elixir
  defmodule D do
    #...
    def get(pid) do
      #...
      Agent.get(pid, fn content -> content end)
      #...
    end
  end
  ```
  
  This spreading of responsibility can generate duplicated code and difficult code maintenance. Besides that, as shown next, due to the lack of control over the format of the shared data, complex composed data can be shared. This freedom to use any format of data is dangerous and can induce developers to introduce bugs.

  ```elixir
  #start an agent with initial state of an empty list
  iex(1)> {:ok, agent} = Agent.start_link fn -> [] end        
  {:ok, #PID<0.135.0>}

  #many data format (i.e., List, Map, Integer, Atom) are
  #combined through direct access spread across the entire system
  iex(2)> A.update(agent)
  iex(3)> B.update(agent)
  iex(4)> C.update(agent)
    
  #state of shared information
  iex(5)> D.get(agent)
  [:atom_value, %{a: 123}]
  ```

* __Refactoring:__ Instead of spreading direct access to an ``Agent`` over many places of the code, it is better to refactor this code by centralizing the responsibility for interacting with an ``Agent`` in a single module. This refactoring improves the maintainability by duplicated code removal and also allows you to limit the accepted format for shared data, reducing bug-proneness. As shown below, the module ``KV.Bucket`` is centralizing the responsibility for interacting with the ``Agent``. Any other place of the code that needs to access shared data must now delegate this action to ``KV.Bucket``. Also, ``KV.Bucket`` now only allows data to be shared in ``Map`` format.

  ```elixir
  defmodule KV.Bucket do
    use Agent

    @doc """
    Starts a new bucket.
    """
    def start_link(_opts) do
      Agent.start_link(fn -> %{} end)
    end

    @doc """
    Gets a value from the `bucket` by `key`.
    """
    def get(bucket, key) do
      Agent.get(bucket, &Map.get(&1, key))
    end

    @doc """
    Puts the `value` for the given `key` in the `bucket`.
    """
    def put(bucket, key, value) do
      Agent.update(bucket, &Map.put(&1, key, value))
    end
  end
  ```

  The following are examples of how to delegate to ``KV.Bucket`` access shared data provided by an ``Agent``.

  ```elixir
  #start an agent through a `KV.Bucket`
  iex(1)> {:ok, bucket} = KV.Bucket.start_link(%{})
  {:ok, #PID<0.114.0>}

  #add shared values to the keys `milk` and `beer`
  iex(2)> KV.Bucket.put(bucket, "milk", 3)
  iex(3)> KV.Bucket.put(bucket, "beer", 7)

  #accessing shared data of specific keys
  iex(4)> KV.Bucket.get(bucket, "beer")   
  7
  iex(5)> KV.Bucket.get(bucket, "milk")
  3
  ```

  These examples are based on codes written in Elixir's official documentation. Source: [link][AgentObsessionExample]

[▲ back to Index](#table-of-contents)
___

### Unsupervised process

* __Category:__ Design-related smell.

* __Problem:__ In Elixir, creating a process outside a supervision tree is not a code smell in itself. However, when a code creates a large number of long-running processes outside a supervision tree, this can turn visibility and monitoring of these processes difficult, therefore not allowing developers to fully control their applications.

* __Example:__ The following code example seeks to illustrate a library responsible for maintaining a numerical ``Counter`` through a ``GenServer`` process outside a supervision tree. Multiple counters can be created simultaneously by a client (one process for each counter), making these unsupervised processes difficult to manage. This can cause problems with the initialization, restart, and shutdown of a system.

  ```elixir
  defmodule Counter do
    use GenServer

    @moduledoc """
      Global counter implemented through a GenServer process
      outside a supervision tree.
    """

    @doc """
      Function to create a counter.
        initial_value: any integer value.
        pid_name: optional parameter to define the process name.
                  Default is Counter.
    """
    def start(initial_value, pid_name \\ __MODULE__)
      when is_integer(initial_value) do
      GenServer.start(__MODULE__, initial_value, name: pid_name)
    end

    @doc """
      Function to get the counter's current value.
        pid_name: optional parameter to inform the process name.
                  Default is Counter.
    """
    def get(pid_name \\ __MODULE__) do
      GenServer.call(pid_name, :get)
    end

    @doc """
      Function to changes the counter's current value.
      Returns the updated value.
        value: amount to be added to the counter.
        pid_name: optional parameter to inform the process name.
                  Default is Counter.
    """
    def bump(value, pid_name \\ __MODULE__) do
      GenServer.call(pid_name, {:bump, value})
      get(pid_name)
    end

    ## Callbacks

    @impl true
    def init(counter) do
      {:ok, counter}
    end

    @impl true
    def handle_call(:get, _from, counter) do
      {:reply, counter, counter}
    end

    def handle_call({:bump, value}, _from, counter) do
      {:reply, counter, counter + value}
    end
  end

  #...Use examples...

  iex(1)> Counter.start(0)
  {:ok, #PID<0.115.0>}

  iex(2)> Counter.get()   
  0

  iex(3)> Counter.start(15, C2)  
  {:ok, #PID<0.120.0>}

  iex(4)> Counter.get(C2)
  15

  iex(5)> Counter.bump(-3, C2)
  12

  iex(6)> Counter.bump(7)
  7
  ```

* __Refactoring:__ To ensure that clients of a library have full control over their systems, regardless of the number of processes used and the lifetime of each one, all processes must be started inside a supervision tree. As shown below, this code uses a ``Supervisor`` <sup>[link][Supervisor]</sup> as a supervision tree. When this Elixir application is started, two different counters (``Counter`` and ``C2``) are also started as child processes of the ``Supervisor`` named ``App.Supervisor``. Both are initialized with zero. By means of this supervision tree, is possible to manage the lifecycle of all child processes (e.g., stopping or restarting each one), improving the visibility of the entire app.

  ```elixir
  defmodule SupervisedProcess.Application do
    use Application

    @impl true
    def start(_type, _args) do
      children = [
        # The counters are Supervisor childs started via Counter.start(0)
        %{
          id: Counter,
          start: {Counter, :start, [0]}
        }, 
        %{
          id: C2,
          start: {Counter, :start, [0, C2]}
        }
      ]

      opts = [strategy: :one_for_one, name: App.Supervisor]
      Supervisor.start_link(children, opts)
    end
  end

  #...Use examples...

  iex(1)> Supervisor.count_children(App.Supervisor)   
  %{active: 2, specs: 2, supervisors: 0, workers: 2}

  iex(2)> Counter.get(Counter)   
  0

  iex(3)> Counter.get(C2)
  0

  iex(4)> Counter.bump(7, Counter)
  7

  iex(5)> Supervisor.terminate_child(App.Supervisor, Counter)
  iex(6)> Supervisor.count_children(App.Supervisor)     
  %{active: 1, specs: 2, supervisors: 0, workers: 2}  #only one active

  iex(7)> Counter.get(Counter)   #Error because it was previously terminated
  ** (EXIT) no process: the process is not alive...

  iex(8)> Supervisor.restart_child(App.Supervisor, Counter)  
  iex(9)> Counter.get(Counter)   #after the restart, this process can be accessed again
  0
  ```

  These examples are based on codes written in Elixir's official documentation. Source: [link][UnsupervisedProcessExample]

[▲ back to Index](#table-of-contents)
___

### Large messages between processes

* __Category:__ Design-related smell.

* __Problem:__ In Elixir, processes run isolatedly and concurrent to other ones. The communication between different processes is performed via message passing. The exchange of messages between processes is not a code smell in itself, however, when a huge structure is sent as a message from one process to another, the sender may become blocked. If these large messages exchanges occur frequently, the prolonged and frequent blocking of processes can cause a system to behave anomalously.

* __Example:__ The following code is composed of two modules which will run in different processes each. As the names suggest, the ``Sender`` module has a function responsible for sending messages from one process to another (i.e., ``send_msg/3``). The ``Receiver`` module has a function to create a process to receive messages (i.e., ``create/0``) and another one to handle the received messages (i.e., ``run/0``). If a huge structure, such as a list with 1_000_000 different values, is sent frequently from ``Sender`` to ``Receiver``, the impacts of this smell could be felt.
  
  ```elixir
  defmodule Receiver do
    @doc """
      Function for receiving messages from processes.
    """
    def run do
      receive do
        {:msg, msg_received} -> msg_received
        {_, _} -> "won't match"
      end
    end

    @doc """
      Create a process to receive a message.
      Messages are received in the run() function of Receiver.
    """
    def create do
      spawn(Receiver, :run, [])
    end
  end
  ```

  ```elixir
  defmodule Sender do
    @doc """
      Function for sending messages between processes.
        pid_receiver: message recipient.
        msg: messages of any type and size can be sent.
        id_msg: used by receiver to decide what to do
                when a message arrives.
                Default is the atom :msg
    """
    def send_msg(pid_receiver, msg, id_msg \\ :msg) do
      send(pid_receiver, {id_msg, msg})
    end
  end
  ```
  
  Examples of large messages between processes:

  ```elixir
  iex(1)> pid = Receiver.create
  #PID<0.144.0>

  #Simulating a message with large content
  iex(2)> msg = %{from: inspect(self()), to: inspect(pid), content: [1, 2, 3..1_000_000]}

  iex(3)> Sender.send_msg(pid, msg)
  {:msg, %{content: [1, 2, 3..1000000], from: "#PID<0.105.0>", to: "#PID<0.144.0>"}}
  ```

  This example is based on a original code by Samuel Mullen. Source: [link][LargeMessageExample]

[▲ back to Index](#table-of-contents)
___

### Complex multi-clause function

* __Category:__ Design-related smell.

* __Problem:__ Using multi-clause functions in Elixir, to group functions of the same name, is not a code smell in itself. However, due to the great flexibility provided by this programming feature, some developers may abuse the number of guard clauses and pattern matchings when defining these grouped functions.

* __Example:__ A recurrent example of abusive use of the multi-clause functions is when we’re trying to mix too much business logic into the function definitions. This makes it difficult to read and understand the logic involved in the functions, which may impair code maintainability. Some developers use documentation mechanisms such as ``@doc`` annotations to compensate for poor code readability, but unfortunately, with a multi-clause function, we can only use these annotations once per function name, particularly on the first or header function. As shown next, all other variations of the function need to be documented only with comments, a mechanism that cannot automate tests, leaving the code bug-proneness.

  ```elixir
  @doc """
    Update sharp product with 0 or empty count
    
    ## Examples

      iex> Namespace.Module.update(...)
      expected result...   
  """
  def update(%Product{count: nil, material: material})
    when name in ["metal", "glass"] do
    # ...
  end

  # update blunt product
  def update(%Product{count: count, material: material})
    when count > 0 and material in ["metal", "glass"] do
    # ...
  end

  # update animal...
  def update(%Animal{count: 1, skin: skin})
    when skin in ["fur", "hairy"] do
    # ...
  end
  ```

  This example is based on a original code by Syamil MJ. Source: [link][MultiClauseExample]

[▲ back to Index](#table-of-contents)
___

### Complex API error handling

* __Category:__ Design-related smell.

* __Problem:__ When a function assumes the responsibility of handling multiple errors returned by the same API endpoint, it can become confusing.

* __Example:__ An example of this code smell is when a function uses the ``case`` control-flow structure to handle these multiple variations of response types from an endpoint. This practice can make the function more complex, long, and difficult to understand, as shown next.

  ```elixir
  def get_customer(customer_id) do
    case get("/customers/#{customer_id}") do
      {:ok, %Tesla.Env{status: 200, body: body}} -> {:ok, body}
      {:ok, %Tesla.Env{body: body}} -> {:error, body}
      {:error, _} = other -> other
    end
  end
  ```

* __Refactoring:__ As shown below, in this situation, instead of using the ``case`` control-flow structure, it is better to delegate the responses handling to a specific function (multi-clause), using pattern matching for each API response variation.

  ```elixir
  def get_customer(customer_id) when is_integer(customer_id) do
    "/customers/" <> customer_id
    |> get()
    |> handle_3rd_party_api_response()
  end

  defp handle_3rd_party_api_response({:ok, %Tesla.Env{status: 200, body: body}}) do 
    {:ok, body}
  end

  defp handle_3rd_party_api_response({:ok, %Tesla.Env{body: body}}) do
    {:error, body}
  end

  defp handle_3rd_party_api_response({:error, _} = other) do 
    other
  end
  ```

  This example is based on code written by Zack <sup>[MrDoops][MrDoops]</sup> and Dimitar Panayotov <sup>[dimitarvp][dimitarvp]</sup>. Source: [link][ComplexErrorHandleExample]

[▲ back to Index](#table-of-contents)
___

### Exceptions for control-flow

* __Category:__ Design-related smell.

* __Problem:__ This smell refers to code that forces developers to handle exceptions for control-flow. Exception handling itself does not represent a code smell, but this should not be the only alternative available to developers to handle an error in client code. When developers have no freedom to decide if an error is exceptional or not, this is considered a code smell.

* __Example:__ An example of this code smell, as shown below, is when a library (e.g. ``MyModule``) forces its clients to use ``try .. rescue`` statements to capture and evaluate errors. This library does not allow developers to decide if an error is exceptional or not in their applications.

  ```elixir
  defmodule MyModule do
    def janky_function(value) do
      if is_integer(value) do
        #...
        "Result..."
      else
        raise RuntimeError, message: "invalid argument. Is not integer!"
      end
    end
  end
  ```

  ```elixir
  defmodule Client do
   
    # Client forced to use exceptions for control-flow.
    def foo(arg) do
      try do
        value = MyModule.janky_function(arg)
        "All good! #{value}."
      rescue
        e in RuntimeError ->
          reason = e.message
          "Uh oh! #{reason}."
      end
    end

  end

  #...Use examples...

  iex(1)> Client.foo(1)                   
  "All good! Result...."

  iex(2)> Client.foo("lucas")
  "Uh oh! invalid argument. Is not integer!."
  ```

* __Refactoring:__ A library author shall guarantee that clients are not required to use exceptions for control-flow in their applications. As shown below, this can be done by refactoring the library ``MyModule``, providing two versions of the function that forces clients to use exceptions for control-flow (e.g., ``janky_function``). 1) a version with the raised exceptions should have the same name as the smelly one, but with a trailing ! (i.e., ``janky_function!``); 2) Another version without raised exceptions should have a name identical to the original version (i.e., ``janky_function``), and should return the result wrapped in a tuple.

  ```elixir
  defmodule MyModule do
    @moduledoc """
      Refactored library
    """
    def janky_function!(value) do
      if is_integer(value) do
        #...
        "Result..."
      else
        raise RuntimeError, message: "invalid argument. Is not integer!"
      end
    end
  
    @doc """
      Alternative version without exceptions for control-flow.
    """
    def janky_function(value) do
      try do
        {:ok, janky_function!(value)}
      rescue
        RuntimeError ->
          {:error, "invalid argument. Is not integer!"}
      end
    end
  end
  ```

  This refactoring gives clients more freedom to decide how to proceed in the event of errors, defining what is exceptional or not in different situations. As shown next, when an error is not exceptional, clients can use specific control-flow structures, such as the ``case`` statement along with pattern matching.

  ```elixir
  defmodule Client do
   
    #Clients now can also choose to use control-flow structures
    #for control-flow when an error is not exceptional.
    def foo(arg) do
      case MyModule.janky_function(arg) do
        {:ok, value} -> "All good! #{value}."
        {:error, reason} -> "Uh oh! #{reason}."
      end
    end

  end

  #...Use examples...

  iex(1)> Client.foo(1)                   
  "All good! Result...."

  iex(2)> Client.foo("lucas")
  "Uh oh! invalid argument. Is not integer!."
  ```

  This example is based on code written by Tim Austin <sup>[neenjaw][neenjaw]</sup> and Angelika Tyborska <sup>[angelikatyborska][angelikatyborska]</sup>. Source: [link][ExceptionsForControlFlowExamples]

[▲ back to Index](#table-of-contents)
___

### Untested polymorphic behavior

* __Category:__ Design-related smell.

* __Problem:__ This code smell refers to functions that have protocol-dependent parameters and are therefore polymorphic. A polymorphic function itself does not represent a code smell, but some developers implement these generic functions without accompanying guard clauses (in this case, to verify whether the parameter types implement the required protocols).

* __Example:__ An instance of this code smell happens when a function uses ``to_string()`` to convert data received by parameter. The function ``to_string()`` uses the protocol ``String.Chars`` for conversions. Many Elixir's data types like ``BitString``, ``Integer``, ``Float``, and ``URI`` implement this protocol. However, as shown below, other Elixir's data types such as ``Map`` do not implement it, thus making the behavior of the ``dasherize/1`` function unpredictable.

  ```elixir
  defmodule CodeSmells do
    def dasherize(data) do
      to_string(data)
      |> String.replace("_", "-")
    end
  end

  #...Use examples...

  iex(1)> CodeSmells.dasherize("Lucas_Vegi")
  "Lucas-Vegi"

  iex(2)> CodeSmells.dasherize(10)
  "10"

  iex(3)> CodeSmells.dasherize(URI.parse("http://www.code_smells.com"))
  "http://www.code-smells.com"

  iex(4)> CodeSmells.dasherize(%{last_name: "vegi", first_name: "lucas"})
  ** (Protocol.UndefinedError) protocol String.Chars not implemented 
  for %{first_name: "lucas", last_name: "vegi"} of type Map
  ```

* __Refactoring:__ There are two main ways to improve code affected by this smell. 1) Write test cases (via ``@doc``) that validate the function for data types that implement the desired protocol; and 2) Implement the function as multi-clause, directing its behavior through guard clauses, as shown below.

  ```elixir
  defmodule CodeSmells do
    @doc """
      Function that converts underscores to dashes.
      Created to illustrate "Untested polymorphic behavior".

      ## Examples

          iex> CodeSmells.dasherize(%{last_name: "vegi", first_name: "lucas"})
          "first-name, last-name"

          iex> CodeSmells.dasherize("Lucas_Vegi")
          "Lucas-Vegi"

          iex> CodeSmells.dasherize(10)
          "10"
    """
    def dasherize(data) when is_map(data) do
      Map.keys(data)
      |> Enum.map(fn key -> "#{key}" end)
      |> Enum.join(", ")
      |> dasherize()
    end

    def dasherize(data) do
      to_string(data)
      |> String.replace("_", "-")
    end
  end

  #...Uses example...

  iex(1)> CodeSmells.dasherize(%{last_name: "vegi", first_name: "lucas"})
  "first-name, last-name"
  ```

  This example is based on code written by José Valim. Source: [link][JoseValimExamples]

[▲ back to Index](#table-of-contents)
___

### Code organization by process

* __Category:__ Design-related smell.

* __Problem:__ This smell refers to codes unnecessarily organized by processes. A process itself does not represent a code smell, but it should only be used to model runtime properties (e.g., concurrency, access to shared resources, and event scheduling). When a process is used for code organization, it can create bottlenecks in the system.

* __Example:__ An example of this code smell, as shown below, is a library that implements arithmetic operations (e.g., add, and subtract) by means of a ``GenSever`` process<sup>[link][GenServer]</sup>. If the number of calls to this single process grows, this code organization can compromise the system performance, therefore becoming a bottleneck.

  ```elixir
  defmodule Calculator do
    use GenServer

    @moduledoc """
      Calculator that performs two basic arithmetic operations.
      This code is unnecessarily organized by a GenServer process.
    """

    @doc """
      Function to perform the sum of two values.
    """
    def add(a, b, pid) do
      GenServer.call(pid, {:add, a, b})
    end

    @doc """
      Function to perform subtraction of two values.
    """
    def subtract(a, b, pid) do
      GenServer.call(pid, {:subtract, a, b})
    end

    def init(init_arg) do
      {:ok, init_arg}
    end

    def handle_call({:add, a, b}, _from, state) do
      {:reply, a + b, state}
    end

    def handle_call({:subtract, a, b}, _from, state) do
      {:reply, a - b, state}
    end
  end

  # Start a generic server process
  iex(1)> {:ok, pid} = GenServer.start_link(Calculator, :init) 
  {:ok, #PID<0.132.0>}

  #...Use examples...
  iex(2)> Calculator.add(1, 5, pid)
  6

  iex(3)> Calculator.subtract(2, 3, pid)
  -1
  ```

* __Refactoring:__ In Elixir, as shown next, code organization must be done only by modules and functions. Whenever possible, a library should not be organized imposing specific behavior such as parallelization to its clients. It is better to delegate this behavioral decision to the developers of clients, thus increasing the potential for code reuse of a library.

  ```elixir
  defmodule Calculator do
    def add(a, b) do
      a + b
    end

    def subtract(a, b) do
      a - b
    end
  end

  #...Use examples...

  iex(1)> Calculator.add(1, 5)
  6

  iex(2)> Calculator.subtract(2, 3)
  -1
  ```
  
  This example is based on code provided in Elixir's official documentation. Source: [link][CodeOrganizationByProcessExample]

[▲ back to Index](#table-of-contents)
___

### Data manipulation by migration

* __Category:__ Design-related smell.

* __Problem:__ This code smell refers to modules that perform both data and structural changes in a database schema via ``Ecto.Migration``<sup>[link][Migration]</sup>. Migrations must be used exclusively to modify a database schema over time (e.g., by including or excluding columns and tables). When this responsibility is mixed data manipulation code, the module becomes low cohesive, more difficult to test, and therefore more bug-proneness.

* __Example:__ An example of this code smell is when an ``Ecto.Migration`` is used simultaneously to alter a table, adding a new column to it, and also to update all pre-existing data in that table, assigning a value to this new column. As shown below, in addition to adding the ``is_custom_shop`` column in the ``guitars`` table, this ``Ecto.Migration`` changes the value of this column for some specific guitar models.

  ```elixir
  defmodule GuitarStore.Repo.Migrations.AddIsCustomShopToGuitars do
    use Ecto.Migration

    import Ecto.Query
    alias GuitarStore.Inventory.Guitar
    alias GuitarStore.Repo

    @doc """
      A function that modifies the structure of table "guitars", 
      adding column "is_custom_shop" to it. By default, all data 
      pre-stored in this table will have the value false stored 
      in this new column.

      Also, this function updates the "is_custom_shop" column value 
      of some guitar models to true.
    """
    def change do
      alter table("guitars") do
        add :is_custom_shop, :boolean, default: false
      end
      create index("guitars", ["is_custom_shop"])
      
      custom_shop_entries()
      |> Enum.map(&update_guitars/1)
    end

    @doc """
      A function that updates values of column "is_custom_shop" to true.
    """
    defp update_guitars({make, model, year}) do
      from(g in Guitar,
        where: g.make == ^make and g.model == ^model and g.year == ^year,
        select: g
      )
      |> Repo.update_all(set: [is_custom_shop: true])
    end

    @doc """
      Function that defines which guitar models that need to have the values 
      of the "is_custom_shop" column updated to true.
    """
    defp custom_shop_entries() do
      [
        {"Gibson", "SG", 1999},
        {"Fender", "Telecaster", 2020}
      ]
    end
  end
  ```

  You can run this smelly migration above by going to the root of your project and typing the next command via console:
  
  ```elixir
    mix ecto.migrate
  ```
  
* __Refactoring:__ To remove this code smell, it is necessary to separate the data manipulation in a ``mix task`` <sup>[link][MixTask]</sup> different from the module that performs the structural changes in the database via ``Ecto.Migration``. This separation of responsibilities is a best practice for increasing code testability. As shown below, the module ``AddIsCustomShopToGuitars`` now use ``Ecto.Migration`` only to perfom structural changes in the database schema:

  ```elixir
  defmodule GuitarStore.Repo.Migrations.AddIsCustomShopToGuitars do
    use Ecto.Migration

    @doc """
      A function that modifies the structure of table "guitars", 
      adding column "is_custom_shop" to it. By default, all data 
      pre-stored in this table will have the value false stored 
      in this new column.
    """
    def change do
      alter table("guitars") do
        add :is_custom_shop, :boolean, default: false
      end

      create index("guitars", ["is_custom_shop"])
    end
  end
  ```

  Furthermore, the new mix task ``PopulateIsCustomShop``, shown next, has only the responsibility to perform data manipulation, thus improving testability:

  ```elixir
  defmodule Mix.Tasks.PopulateIsCustomShop do
    @shortdoc "Populates is_custom_shop column"

    use Mix.Task

    import Ecto.Query
    alias GuitarStore.Inventory.Guitar
    alias GuitarStore.Repo

    @requirements ["app.start"]

    def run(_) do
      custom_shop_entries()
      |> Enum.map(&update_guitars/1)
    end

    defp update_guitars({make, model, year}) do
      from(g in Guitar,
        where: g.make == ^make and g.model == ^model and g.year == ^year,
        select: g
      )
      |> Repo.update_all(set: [is_custom_shop: true])
    end

    defp custom_shop_entries() do
      [
        {"Gibson", "SG", 1999},
        {"Fender", "Telecaster", 2020}
      ]
    end
  end
  ```  

  You can run this ``mix task`` above typing the next command via console:
  
  ```elixir
    mix populate_is_custom_shop
  ```
  
  This example is based on code originally written by Carlos Souza. Source: [link][DataManipulationByMigrationExamples]

[▲ back to Index](#table-of-contents)

## Low-level concerns smells

Low-level concerns smells are more simple than design-related smells and affect a small part of the code. Next, all 8 different smells classified as low-level concerns are explained and exemplified:

### Working with invalid data

TODO...

[▲ back to Index](#table-of-contents)
___

### Map/struct dynamic access

TODO...

[▲ back to Index](#table-of-contents)
___

### Unplanned value extraction

TODO...

[▲ back to Index](#table-of-contents)
___

### Modules with identical names

TODO...

[▲ back to Index](#table-of-contents)
___

### Unnecessary macro

TODO...

[▲ back to Index](#table-of-contents)
___

### App configuration for code libs

TODO...

[▲ back to Index](#table-of-contents)
___

### Compile-time app configuration

TODO...

[▲ back to Index](#table-of-contents)
___

### Dependency with "use" when an "import" is enough

TODO...

[▲ back to Index](#table-of-contents)

## About

This catalog, composed of 18 code smells specific for the [Elixir programming language][Elixir], was originally proposed by Lucas Vegi and Marco Tulio Valente ([ASERG/DCC/UFMG][ASERG]).

To better organize the impacts caused by these smells, we classify them into two different groups: __design-related smells__ <sup>[link](#design-related-smells)</sup> and __low-level concerns smells__ <sup>[link](#low-level-concerns-smells)</sup>.

Please feel free to make pull requests and suggestions ([Discussions][Discussions] tab).

[▲ back to Index](#table-of-contents)

<!-- Links -->
[Elixir Smells]: https://github.com/lucasvegi/Elixir-Code-Smells
[Elixir]: http://elixir-lang.org
[ASERG]: http://aserg.labsoft.dcc.ufmg.br/
[MultiClauseExample]: https://syamilmj.com/2021-09-01-elixir-multi-clause-anti-pattern/
[ComplexErrorHandleExample]: https://elixirforum.com/t/what-are-sort-of-smells-do-you-tend-to-find-in-elixir-code/14971
[JoseValimExamples]: http://blog.plataformatec.com.br/2014/09/writing-assertive-code-with-elixir/
[dimitarvp]: https://elixirforum.com/u/dimitarvp
[MrDoops]: https://elixirforum.com/u/MrDoops
[neenjaw]: https://exercism.org/profiles/neenjaw
[angelikatyborska]: https://exercism.org/profiles/angelikatyborska
[ExceptionsForControlFlowExamples]: https://exercism.org/tracks/elixir/concepts/try-rescue
[DataManipulationByMigrationExamples]: https://www.idopterlabs.com.br/post/criando-uma-mix-task-em-elixir
[Migration]: https://hexdocs.pm/ecto_sql/Ecto.Migration.html
[MixTask]: https://hexdocs.pm/mix/Mix.html#module-mix-task
[CodeOrganizationByProcessExample]: https://hexdocs.pm/elixir/master/library-guidelines.html#avoid-using-processes-for-code-organization
[GenServer]: https://hexdocs.pm/elixir/master/GenServer.html
[UnsupervisedProcessExample]: https://hexdocs.pm/elixir/master/library-guidelines.html#avoid-spawning-unsupervised-processes
[Supervisor]: https://hexdocs.pm/elixir/master/Supervisor.html
[Discussions]: https://github.com/lucasvegi/Elixir-Code-Smells/discussions
[LargeMessageExample]: https://samuelmullen.com/articles/elixir-processes-send-and-receive/
[Agent]: https://hexdocs.pm/elixir/1.13/Agent.html
[Task]: https://hexdocs.pm/elixir/1.13/Task.html
[GenServer]: https://hexdocs.pm/elixir/1.13/GenServer.html
[AgentObsessionExample]: https://elixir-lang.org/getting-started/mix-otp/agent.html#agents
[ElixirInProduction]: https://elixir-companies.com/
