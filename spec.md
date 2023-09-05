The base DAP spec can be found [here](https://microsoft.github.io/debug-adapter-protocol/specification).


Our implementation uses a subset of the DAP's requests and events, while adding
some new ones.

From here on, "backend" refers to the debugger process itself, and "frontend"
refers to the interface/IDE/etc.

Requests flow from frontend to backend, with a (potentially empty) response to the
frontend.
Events flow from backend to frontend.

---

<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
**Table of Contents**

- [A note on multiple procs](#a-note-on-multiple-procs)
- [Types](#types)
    - [`ExecMap`](#execmap)
    - [`UnifyMap`](#unifymap)
    - [Debugger state](#debugger-state)
- [Events & Requests](#events--requests)
    - [Lifecycle](#lifecycle)
        - [Events](#events)
            - [Initialized](#initialized)
        - [Requests](#requests)
            - [Initialize](#initialize)
            - [ConfigurationDone](#configurationdone)
            - [`+` Launch](#-launch)
            - [Attach](#attach)
            - [Disconnect](#disconnect)
    - [Inspection](#inspection)
        - [Requests](#requests-1)
            - [Threads](#threads)
            - [StackTrace](#stacktrace)
            - [`~` Scopes](#-scopes)
            - [Variables](#variables)
            - [ExceptionInfo](#exceptioninfo)
            - [`*` DebuggerState](#-debuggerstate)
            - [`*` Unification](#-unification)
        - [Events](#events-1)
            - [`*` Log](#-log)
            - [`*` DebugStateUpdate](#-debugstateupdate)
    - [Controlling execution](#controlling-execution)
        - [Requests](#requests-2)
            - [`~` Continue](#-continue)
            - [`~` Next](#-next)
            - [ReverseContinue](#reversecontinue)
            - [StepBack](#stepback)
            - [`~` StepIn](#-stepin)
            - [`~` StepOut](#-stepout)
            - [`*` Jump](#-jump)
            - [`*` StepSpecific](#-stepspecific)
            - [`*` StartProc](#-startproc)
        - [Events](#events-2)
            - [Stopped](#stopped)

<!-- markdown-toc end -->

---

## A note on multiple procs
The debugger is intended to operates on one procedure.
However, sometimes parts of a procedure - for example, the body of a while loop -
are abstracted away as other functions. This is why the protocol is built to
support multiple simultaneous procs.

# Types

Extracted from `types.d.ts` in the debugger extension. For context, these types
have largely been formed from the JSON representation of the relevant OCaml types
given by yojson.

All these types should be considered immutable; `readonly` statements have been
removed for readability's sake.

<details open>
<summary><i>Expand/Contract</i></summary>

## `ExecMap`
```ts
type BranchCase = {
  kind: string;
  display: [string, string];
  json: any;
};

type Submap =
  | ['NoSubmap']
  | ['Submap', ExecMap]
  | ['Proc', string];

type Unification = {
  id: number;
  kind: UnifyKind;
  result: UnifyResult;
};

type CmdData = {
  ids: number[];
  display: string;
  unifys: Unification[];
  errors: string[];
  submap: Submap;
};

type ExecMap =
  | ['Nothing']
  | ['Cmd', { data: CmdData; next: ExecMap }]
  | [
      'BranchCmd',
      { data: CmdData; nexts: [BranchCase, [null, ExecMap]][] }
    ]
  | ['FinalCmd', { data: CmdData }];
```

## `UnifyMap`
```ts
type UnifyResult = ['Success' | 'Failure'];

type Substitution = {
  assertId: number;
  subst: [string, string];
};

type AssertionData = {
  id: number;
  fold: [number, UnifyResult] | null;
  assertion: string;
  substitutions: Substitution[];
};

type UnifySeg =
  | ['Assertion', AssertionData, UnifySeg]
  | ['UnifyResult', number, UnifyResult];

type UnifyKind = [
  'Postcondition' | 'Fold' | 'FunctionCall' | 'Invariant' | 'LogicCommand'
];

type UnifyMapInner =
  | ['Direct', UnifySeg]
  | ['Fold', UnifySeg[]];

type UnifyMap = [UnifyKind, UnifyMapInner];
```

## Debugger state
```ts
type DebugProcState = {
  execMap: ExecMap;
  liftedExecMap: ExecMap | null;
  currentCmdId: number;
  unifys: Unification[];
  procName: string;
};

type DebuggerState = {
  mainProc: string;
  currentProc: string;
  procs: Record<string, DebugProcState>;
};
```

</details>

# Events & Requests

- `~` means the request/event hasn't been modified, but the behavior is different
  (or not immediately apparent) in symbolic debugging
- `+` means the request/event has been extended in some way
- `*` denotes a request/event new for the SEDAP

## Lifecycle

<details open>
<summary><i>Expand/Contract</i></summary>

### Events

#### Initialized

> This event indicates that the debug adapter is ready to accept configuration
> requests (e.g. `setBreakpoints`, `setExceptionBreakpoints`).
> 
> A debug adapter is expected to send this event when it is ready to accept
> configuration requests (but not before the `initialize` request has finished).
> 
> The sequence of events/requests is as follows:
> 
> - adapters sends `initialized` event (after the initialize request has
>   returned)
> - client sends zero or more `setBreakpoints` requests
> - client sends one `setFunctionBreakpoints` request (if corresponding
>   capability `supportsFunctionBreakpoints` is true)
> - client sends a `setExceptionBreakpoints` request if one or more
>   `exceptionBreakpointFilters` have been defined (or if
>   `supportsConfigurationDoneRequest` is not true)
> - client sends other future configuration requests
> - client sends one `configurationDone` request to indicate the end of the
>   configuration.

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Events_Initialized)

### Requests

#### Initialize

> The `initialize` request is sent as the first request from the client to the
> debug adapter in order to configure it with client capabilities and to retrieve
> capabilities from the debug adapter.

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_Initialize)

#### ConfigurationDone

> This request indicates that the client has finished initialization of the debug
> adapter.
> 
> So it is the last request in the sequence of configuration requests (which was
> started by the `initialized` event).

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_ConfigurationDone)

#### `+` Launch

> This `launch` request is sent from the client to the debug adapter to start the
> debuggee with or without debugging (if `noDebug` is true).
> 
> Since launching is debugger/runtime specific, the arguments for this request are
> not part of this specification.

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_Launch)

```ts
interface LaunchRequest extends Request { 
  type: 'launch';
  
  arguments: {
    noDebug?: boolean;
    __restart?: any;
    
    // Gillian-specific arguments
    program: string;
    procedure_name?: string;
    stop_on_entry: boolean;  // Akin to putting a breakpoint on the first command
  };
}
```

#### Attach

⚠ ️**Not supported by the Gillian debugger** ⚠️

> The `attach` request is sent from the client to the debug adapter to attach to a
> debuggee that is already running.
>
> Since attaching is debugger/runtime specific, the arguments for this request
> are not part of this specification.

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_Attach)


#### Disconnect

⚠ ️**Not supported by the Gillian debugger** ⚠️

> The `disconnect` request asks the debug adapter to disconnect from the debuggee
> (thus ending the debug session) and then to shut down itself (the debug
> adapter).
> 
> In addition, the debug adapter must terminate the debuggee if it was started
> with the `launch` request. If an `attach` request was used to connect to the
> debuggee, then the debug adapter must not terminate the debuggee.

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_Disconnect)

</details>

## Inspection

<details open>
<summary><i>Expand/Contract</i></summary>

### Requests

#### Threads

> The request retrieves a list of all threads.

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_Threads)

#### StackTrace

> The request returns a stacktrace from the current execution state of a given
> thread.

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_StackTrace)

#### `~` Scopes

> The request returns the variable scopes for a given stack frame ID.

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_Scopes)

For Gillian verification, instead of showing actual variable scopes (which don't
really come into play in per-function verification), this is used to show the
symbolic state, with each part of the state given a "scope" in the context of
this request.

The given scopes are, generally speaking:
- **Store**: the (potentially symbolic) values of program variables
- **Memory**: assertions about the contents of memory (though some memory
  information may be captured by predicates and not shown here)
- **Pure Formulae**: pure logical formulae
- **Typing Environment**: the types of logical variables
- **Predicates**: the predicates that currently hold

#### Variables

> Retrieves all child variables for the given variable reference.

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_Variables)

#### ExceptionInfo

> Retrieves the details of the exception that caused this event to be raised.

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_ExceptionInfo)

#### `*` DebuggerState

Requests the current debugging state, i.e. any information that isn't already
communicated by the base DAP.

```ts
interface DebuggerStateRequest extends Request {
  type 'debuggerState';

  arguments: {};
};

interface DebuggerStateResponse extends Response {
  body: DebuggerState;
};
```

#### `*` Unification

Requests the map of a unification.

```ts
interface UnificationRequest extends Request {
  type: 'unification';

  arguments: {
    id: number;
  };
};

interface UnificationResponse extends Response {
  body: {
    unifyId: number;
    unifyMap: UnifyMap;
  };
};
```

### Events

#### `*` Log

Logs a message to the frontend, (optionally) along with some JSON.

This may be not be necessary given the presence of the
[`output` event](https://microsoft.github.io/debug-adapter-protocol/specification#Events_Output)
from the base DAP.

```ts
interface LogEvent extends Event {
  event: 'log';

  body: {
    msg: string;
    json: any;
  };
};
```

#### `*` DebugStateUpdate
This is sent to update the frontend on any information on the state of debugging
that isn't already communicated by the base DAP.

This generally sent alongside the [`stopped` event](#stopped).

```ts
interface DebugStateUpdateEvent extends Event {
  event: 'debugStateUpdate';

  body: DebuggerState;
};
```

</details>


## Controlling execution

<details open>
<summary><i>Expand/Contract</i></summary>

### Requests

#### `~` Continue

> The request resumes execution [...].

This behaves largely as expected in symbolic debugging, though worth noting:
all unexplored branches *from the current command onwards* will be explored,
either until termination (via return or error) or a breakpoint.

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_Continue)

#### `~` Next

> The request executes one step (in the given granularity) [...].

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_Next)

This behaves as expected, though if stepping forwards from a branching command,
the "first" branch is selected. This can be decided arbitrarily, so using the
[`StepSpecific` request](#-stepspecific) is preferred.

#### ReverseContinue

> The request resumes backward execution [...].

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_ReverseContinue)

Since branching only occurs forwards, backwards execution behaves as expected.

#### StepBack

> The request executes one backward step (in the given granularity) [...].

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_ReverseContinue)

As with [ReverseContinue](#reversecontinue).

#### `~` StepIn

❓ **This request's behaviour is not fully defined in the Gillian debugger** ❓

> The request resumes the given thread to step into a function/method [...].
> 
> [...]
>
> If the request cannot step into a target, `stepIn` behaves like the `next`
> request.

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_StepIn)

This may be used to step into sub-procedure execution -
[see above](#a-note-on-multiple-procs).

#### `~` StepOut

❓ **This request's behaviour is not fully defined in the Gillian debugger** ❓

> The request resumes the given thread to step out (return) from a
> function/method [...].

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Requests_StepOut)

This may be used to step out of sub-procedure execution -
[see above](#a-note-on-multiple-procs).

#### `*` Jump
"Jumps" to a point in execution. This can be before, after or on a different
branch from the current command.

```ts
interface JumpRequest extends Request {
  type: 'jump';

  arguments: {
    id: number;
  };
};

interface JumpResponse extends Response {
  body: {
    success: bool;
    err: string | null;
  };
};
```


#### `*` StepSpecific

Steps forward from a specific command, in a specific branch case (if that command
branches). Think of this as [`jump`](#jump) followed by [`next`](#next), but
specifying a branch case if relevant.

```ts
interface StepSpecificRequest extends Request {
  type: 'stepSpecific';

  arguments: {
    prevId: string;
    branchCase: BranchCase | null;
  };
};

interface StepSpecificResponse extends Response {
  body: {
    success: bool;
    err: string | null;
  };
};
```

#### `*` StartProc
Used to initiate sub-procedures - [see above](#a-note-on-multiple-procs).

This can potentially be removed in favour of automatically starting
sub-procedures as necessary when `stepIn` is used.

```ts
interface StartProcRequest extends Request {
  type: 'startProc';

  arguments: {
    procName: string;
  };
};

interface StartProcResponse extends Response {
  body: {
    success: bool;
    err: string | null;
  };
};
```

### Events

#### Stopped

> The event indicates that the execution of the debuggee has stopped due to some
> condition.
>
> This can be caused by a breakpoint previously set, a stepping request has
> completed, by executing a debugger statement etc.

[*See DAP spec*](https://microsoft.github.io/debug-adapter-protocol/specification#Events_Stopped)

The [`DebuggerStateUpdate` event](#-debugstateupdate) is usually sent alongside
this event.

</details>
