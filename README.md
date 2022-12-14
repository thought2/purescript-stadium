Under construction... <img
src="https://media.tenor.com/MRCIli40TYoAAAAi/under-construction90s-90s.gif" width="30">

# purescript-stadium
Type safe state machines 

## Usage
### Imports
We'll need those imports for the following examples: 
```hs
module Test.Readme where

import Prelude

import Data.Generic.Rep (class Generic)
import Data.Show.Generic (genericShow)
import Data.Variant (Variant)
import Data.Variant as V
import Effect (Effect)
import Node.ChildProcess as CP
import Node.Encoding as Enc
import Node.FS.Sync as FS
import Node.Path (FilePath)
import Stadium (type (>>), At, Cases)
import Stadium as SD
import Stadium.DotGraph as SD.G
import Stadium.Reflection (Proxy3(..), ValidProtocol)
import Stadium.Reflection as SD.R
import Type.Proxy (Proxy(..))
```
### A simple conventional state machine using ADT's
Let's first look at an example state machine implementation which is done
without any extra library.
It consists of a type for the state, a type for a message and a reducer
function that is can produce new states.

```hs
data State'1 = On | Off

data Msg'1 = SwitchOn | SwitchOff

derive instance Generic State'1 _
derive instance Eq State'1

instance Show State'1 where
  show = genericShow

reducer'1 :: Msg'1 -> State'1 -> State'1
reducer'1 msg state = case msg of
  SwitchOn -> case state of
    Off -> On
    _ -> state
  SwitchOff -> case state of
    On -> Off
    _ -> state
```
We could use it like so:
```hs
state'1 :: State'1
state'1 = Off
  # reducer'1 SwitchOn
  # reducer'1 SwitchOff
  # reducer'1 SwitchOff
  # reducer'1 SwitchOn
```
What's the problem with this approach? If you look at the reducer function
you'll notice that it's quite easy to introduce mistakes. For instance the
`SwitchOn` handler could mistakenly return the `Off` state. The types are not
strict enough to prevent this. It may look silly in this simple example but
state machines can become quite large. And then it will be useful to
have types that describe the protocol of a state machine more accurately. 

### A state machine with Variant types
Before we move to the first example for this library, let's quickly redefine the same machine
with a powerful sum type alternative called [variant
types](https://github.com/natefaubion/purescript-variant). I recommend to get
familiar with this library by reading its README. We'll see later that we
don't necessarily need to use Variant types in our state type. However,
they're inevitably a part of our API and using them is a good way get a
better understanding of this library.
```hs
type State'2 = Variant
  ( on :: Unit
  , off :: Unit
  )

type Msg'2 = Variant
  ( switchOn :: Unit
  , switchOff :: Unit
  )

reducer'2 :: Msg'2 -> State'2 -> State'2
reducer'2 msg state =
  msg #
    ( V.case_ # V.onMatch
        { switchOn: \_ ->
            state #
              ( V.default state #
                  V.onMatch
                    { off: \_ -> V.inj (Proxy :: _ "on") unit
                    }
              )

        , switchOff: \_ ->
            state #
              ( V.default state #
                  V.onMatch
                    { on: \_ -> V.inj (Proxy :: _ "off") unit
                    }
              )
        }
    )
```
Above we defined the types using Variants. The `Unit` type is used when a
case does not hold any data. The reducer function looks a bit complex. Right
now, it has the same behavior as the one in the ADT example.

First we do an exhaustive pattern match on the incoming message using the
`case_` function. Inside each arm we do a non exhaustive pattern match on
the incoming state providing a default value as fallback. The
injection syntax for variant types is a bit wordy, too. This will be simpler once
PureScript has "visible type application", which as of the time of writing is
being worked on. Until then, you can use the
[variant-ctors](https://github.com/thought2/purescript-variant-ctors) package
to generate constructors.

To be complete, here's a usage example of this implementation, too:
```hs
state'2 :: State'2
state'2 = V.inj (Proxy :: _ "off") unit
  # reducer'2 (V.inj (Proxy :: _ "switchOn") unit)
  # reducer'2 (V.inj (Proxy :: _ "switchOff") unit)
  # reducer'2 (V.inj (Proxy :: _ "switchOff") unit)
  # reducer'2 (V.inj (Proxy :: _ "switchOn") unit)
```
### Increase type safety by using `stadium` 
Let's see how we can improve the previous example. We'll use the same state
and message type.

<img src="assets/state-3.svg"/>

```hs
type State'3 = State'2

type Msg'3 = Msg'2
```
However, now we define a protocol specification for our state machine. It
indicates that the message "switchOff" is only allowed to turn the state from
"off" into "on". And vice versa for "switchOn". For now, each entry in the
protocol can be understand as a mapping from state cases to state cases.
```hs
type Protocol'3 = SD.Protocol_
  ( switchOff :: Cases (on :: At) >> Cases (off :: At)
  , switchOn :: Cases (off :: At) >> Cases (on :: At)
  )
```
The reducer is quite similar as the previous one but it has an important difference.
```hs
reducer'3 :: Msg'3 -> State'3 -> State'3
reducer'3 = SD.mkReducer
  (Proxy :: _ Protocol'3)
  { switchOn: \_ -> -- wildcard for message (unit)

      V.case_ #
        V.onMatch
          { off: \_ -> -- wildcard for state (unit)

              V.inj (Proxy :: _ "on") unit
          }
  , switchOff: \_ ->
      V.case_ #
        V.onMatch
          { on: \_ ->
              V.inj (Proxy :: _ "off") unit
          }
  }
```
We're using the `mkReducer` function to which we provide our protocol via a
Proxy as the first argument. In the second argument we define the message
handlers inside a record. Thus we already get the outer pattern matching for
free. The inner pattern matching is now exhaustive. We can only match on the
input state that we defined in the protocol for each message. Moreover, the
return type of each branch is only the state that we defined as output above.
In other words, for this simple example, even if we tried there would be no
way to provide a wrong implementation for a case in the reducer.

Of course it is a bit silly to pattern match on a single case, and also to
inject a case into a variant that has only one case. So we can simplify the
example by using some helper functions. Later we'll see that those kind of
one-to-one relations are not always the case, so knowing the verbose syntax
will be helpful soon.
```hs
reducer'3a :: Msg'3 -> State'3 -> State'3
reducer'3a = SD.mkReducer
  (Proxy :: _ Protocol'3)
  { switchOn: \_ -> SD.oneToOne \_ -> unit
  , switchOff: \_ -> SD.oneToOne \_ -> unit
  }
```
### Countdown

<img src="assets/state-4.svg"/>

Finally let's look at a bit more complex example. Here's the complete code
for a countdown state machine:
```hs
type State'4 = Variant
  ( idle :: Unit
  , counting :: Int
  , zero :: Unit
  )

type Msg'4 = Variant
  ( start :: Int
  , countDown :: Unit
  )

type Protocol'4 = SD.Protocol_
  ( start ::
      Cases (idle :: At) >> Cases (counting :: At)

  , countDown ::
      Cases (counting :: At) >> Cases (counting :: At, zero :: At)
  )

reducer'4 :: Msg'4 -> State'4 -> State'4
reducer'4 = SD.mkReducer
  (Proxy :: _ Protocol'4)
  { start: \n ->
      SD.oneToOne \_ -> n

  , countDown: \_ ->
      SD.fromOne \n ->
        if n == 0 then
          V.inj (Proxy :: _ "zero") unit
        else
          V.inj (Proxy :: _ "counting") (n - 1)
  }
```
What's new here is that we now have a "countDown" message that goes from one possible state case
to two possible state cases. That's why we cannot use the the `oneToOne`
helper in ths reducer. We can only use the `fromOne` helper to eliminate the
single case variant input.
```hs

```
### Generating state diagrams

You may have wondered how the graphs of the state diagrams have been created.
The advantage of the approach of making the protocol of the state machine
explicit is that we can turn this information into other representations.
This would not have been possible in the conventional ADT example because all
the wiring is implicitly defined inside the reducer function and there is no
way to access this data from the outside.

The stadium library includes a simple [graphiz](https://graphviz.org/) dot file renderer.
In our case we want to turn a valid state machine protocol into both a ".dot"
file and a ".svg" file.

For this purpose we define the following helper function: 
```hs
writeGraphFiles :: FilePath -> String -> ValidProtocol -> Effect Unit
writeGraphFiles dir name ptc = do
  let
    pathDot = dir <> "/" <> name <> "." <> "dot"
    pathSvg = dir <> "/" <> name <> "." <> "svg"

  SD.G.toDotGraph ptc
    # SD.G.toText
    # FS.writeTextFile Enc.UTF8 pathDot

  void $ CP.spawn "dot" [ "-Tsvg", pathDot, "-o", pathSvg ]
    CP.defaultSpawnOptions
```
Which we can now use like so:
```hs
sampleMain :: Effect Unit
sampleMain = do
  (Proxy3 :: _ Protocol'3 Msg'3 State'3)
    # SD.R.mkValidProtocol
    # writeGraphFiles "assets" "state-3"

  (Proxy3 :: _ Protocol'4 Msg'4 State'4)
    # SD.R.mkValidProtocol
    # writeGraphFiles "assets" "state-4"
```
