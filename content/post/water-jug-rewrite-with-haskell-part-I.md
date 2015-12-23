+++
date = "2015-12-21T22:46:18+05:30"
draft = false
title = "Water Jug Rewrite With Haskell Part I"

+++

## History
Water jug problem is a famous problem commonly found in AI texts. There are few different version of it like these:

1. [cis.udel.edu](https://www.eecis.udel.edu/~mccoy/courses/cisc4-681.10f/lec-materials/handouts/search-water-jug-handout.pdf)
2. [math.tamu.edu](http://www.math.tamu.edu/~dallen/hollywood/diehard/diehard.htm)

I also had to do the same problem in my own college days. At the time I was learning python and thought it might be an interesting problem to solve with python. Looking at the code now I feel embarrassed. Am not even sure how I got it work. Rewriting it in python seems rather uninteresting now. Rather using a static functional language like Haskell seamed to make it more interesting. Obviously once you look at the code you will realize currently I don't know much of Haskell either.

Rather than trying to solve the problem in one shot, I decided to limit the initial version some what.

## Updated Problem

There can be some Water Jars, with any given capacity. The challenge will be to start from that to reach the given final state which will be get some specified quantity of water in these Jars. These Jugs have no measurements. But you know the full capacity of the Jugs.

As a starting point these limitation will be put in place.

1. Max number of Jars will be limited to two.
2. We will only be finding the relevant states the jars can be in and where they can go from these states.

## Data

One of the most interesting things about writing in a language like Haskell is how Types and data become powerful design tools. So to start with let us define `Jug` and `State`  which is the combination of `Jug`s.

```haskell

-- WaterJug.hs

--  Jug capacity, holding
data Jug = Jug Int Int deriving (Show, Eq, Ord)

-- State left right
data State = State Jug Jug deriving (Show, Eq, Ord)

empty :: Int -> Jug
empty c = Jug c 0

```

The `deriving` ensures that we can have reasonable string representation for these objects, they can be equated to each other, ordered etc.

Both these types are tuples. They can only contain two arguments.

`empty` function can be use to create an empty `Jug` of any given capacity.

## Possible Operations on Given Jugs

Given two `Jug`s you can do 6 operations. Depending on what state each `Jug`s are in only some of these operations will be valid at any given state.

### The operations are

* given a `Jug` that has liquid less that full, it can be made full.

``` haskell
forRightFull :: State -> Maybe State
forRightFull (State (Jug rc rh) lj)
  | rc <= rh = Nothing
  | otherwise = Just $ State (Jug rc rc) lj

```

* Given a `Jug` that is non empty and the other `Jug` has some space left, Some of the liquid can be poured from this to the other one.

``` haskell
forRightToLeft :: State -> Maybe State
forRightToLeft (State (Jug rc rh) (Jug lc lh))
  | rh == 0 || lc <= lh = Nothing
  | otherwise = Just $ State (Jug rc (rh -liquidToTransfer)) (Jug lc (lh + liquidToTransfer))
    where
      liquidToTransfer = if maxCanPour >= rh then rh else maxCanPour
      maxCanPour = lc - lh
```

* Given a non empty `Jug` it can be emptied.

``` haskell
forRightToEmpty :: State -> Maybe State
forRightToEmpty (State (Jug rc rh) lj)
  | rh == 0 = Nothing
  | otherwise = Just $ State (empty rc) lj

```

* These operations can be replicated on second `Jug` as is except the `Jug`s should be reversed. `interchange` is a function to help with that.

``` haskell
interchange :: (State -> Maybe State) -> State -> Maybe State
interchange f (State rj lj) =
  case f (State lj rj) of
    Nothing -> Nothing
    Just (State rj' lj') -> Just (State lj' rj')
```

* operations for the left written using operations for right and `interchange`.

``` haskell
forLeftFull :: State -> Maybe State
forLeftFull = interchange forRightFull

forLeftToRight :: State -> Maybe State
forLeftToRight = interchange forRightToLeft

forLeftToEmpty :: State -> Maybe State
forLeftToEmpty = interchange forRightToEmpty
```

Few interesting things to note here. All the action functions have the same signature. `interchange` takes a function as first parameter and we are partially applying it giving us the exact type signature we required. There common type allows us to put all these functions in a  `list`. An important feature we will be using next. We use `option`(`Maybe`) type which allows us denote irrelevant actions on given state.

## Compute Next States

Given any `State`, you can apply all these functions to get all the possible next `State` we can take the Jars to. I ended up doing that like so:

``` haskell
getNextState :: State -> [State]
getNextState s = catMaybes $ map (\f -> f s) toNextStates
  where
    toNextStates = [forRightToLeft, forRightToEmpty, forLeftFull,
                    forLeftToRight, forLeftToEmpty, forRightFull]
```

`toNextStates` is a list of the functions that were defined in the previous section. The only way all these functions can be put in a single list is if they have the same signature.

`catMaybes` filters out all the `Nothing`s and takes out the `Just` values. Thus leaving us with actual/relevant changes.

## Finding relevent states

Now given a starting state and final state, we try to find all the possible states between them. To be able to make the function recursive I have had to make the function accept lot more arguments though. In all probability it will be called by another function, and most of the other parameters can be derived from the first two.

``` haskell
allStates :: State -> State -> S.Set State -> S.Set State -> M.Map State [State] -> M.Map State [State]
allStates i f complete queue m
  | S.null queue = m
  | otherwise = allStates i' f complete' queue'' m'
  where ns = getNextState i
        m' = M.insert i ns m
        queue' = S.delete i queue
        new_q = S.fromList $ filter (\s -> S.notMember s complete) ns
        queue'' = if S.member f new_q then queue' else S.union queue' new_q
        i' = head $ S.toList queue''
        complete' = S.insert i complete
```

You can think of the `where` clauses as being executed top to bottom. In reality though they are all lazy and could be executed as an when the need arises. So order has no meaning.

I am not happy with the number of arguments this function takes, but currently cant think of a better way.

## Conclusion

To conclude for now let me present a simple REPL session.

``` haskell
> :l WaterJug.hs
> import qualified Data.Map as M
> import Data.Maybe
> import qualified Data.Set as Set
> allStates (State (emptyJug 4) (emptyJug 3)) (State (Jug 4 2) (emptyJug 3)) Set.empty (Set.fromList [(State (Jug 4 4) (emptyJug 3))]) M.empty
fromList [(State (Jug 4 0) (Jug 3 0),[State (Jug 4 0) (Jug 3 3),State (Jug 4 4) (Jug 3 0)]),(State (Jug 4 0) (Jug 3 1),[State (Jug 4 0) (Jug 3 3),State (Jug 4 1) (Jug 3 0),State (Jug 4 0) (Jug 3 0),State (Jug 4 4) (Jug 3 1)]),(State (Jug 4 0) (Jug 3 2),[State (Jug 4 0) (Jug 3 3),State (Jug 4 2) (Jug 3 0),State (Jug 4 0) (Jug 3 0),State (Jug 4 4) (Jug 3 2)]),(State (Jug 4 0) (Jug 3 3),[State (Jug 4 3) (Jug 3 0),State (Jug 4 0) (Jug 3 0),State (Jug 4 4) (Jug 3 3)]),(State (Jug 4 1) (Jug 3 0),[State (Jug 4 0) (Jug 3 1),State (Jug 4 0) (Jug 3 0),State (Jug 4 1) (Jug 3 3),State (Jug 4 4) (Jug 3 0)]),(State (Jug 4 1) (Jug 3 3),[State (Jug 4 0) (Jug 3 3),State (Jug 4 4) (Jug 3 0),State (Jug 4 1) (Jug 3 0),State (Jug 4 4) (Jug 3 3)]),(State (Jug 4 2) (Jug 3 3),[State (Jug 4 0) (Jug 3 3),State (Jug 4 4) (Jug 3 1),State (Jug 4 2) (Jug 3 0),State (Jug 4 4) (Jug 3 3)]),(State (Jug 4 3) (Jug 3 0),[State (Jug 4 0) (Jug 3 3),State (Jug 4 0) (Jug 3 0),State (Jug 4 3) (Jug 3 3),State (Jug 4 4) (Jug 3 0)]),(State (Jug 4 3) (Jug 3 3),[State (Jug 4 0) (Jug 3 3),State (Jug 4 4) (Jug 3 2),State (Jug 4 3) (Jug 3 0),State (Jug 4 4) (Jug 3 3)]),(State (Jug 4 4) (Jug 3 0),[State (Jug 4 1) (Jug 3 3),State (Jug 4 0) (Jug 3 0),State (Jug 4 4) (Jug 3 3)]),(State (Jug 4 4) (Jug 3 1),[State (Jug 4 2) (Jug 3 3),State (Jug 4 0) (Jug 3 1),State (Jug 4 4) (Jug 3 3),State (Jug 4 4) (Jug 3 0)]),(State (Jug 4 4) (Jug 3 2),[State (Jug 4 3) (Jug 3 3),State (Jug 4 0) (Jug 3 2),State (Jug 4 4) (Jug 3 3),State (Jug 4 4) (Jug 3 0)]),(State (Jug 4 4) (Jug 3 3),[State (Jug 4 0) (Jug 3 3),State (Jug 4 4) (Jug 3 0)])]
>
```
