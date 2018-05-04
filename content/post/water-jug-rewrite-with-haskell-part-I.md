+++
date = "2018-05-04T22:46:18+05:30"
draft = false
title = "Water Jug Rewrite With Haskell Part I"

+++

## History
Water jug problem is a famous problem commonly found in AI texts. There are few different version of it like these:

1. [cis.udel.edu](https://www.eecis.udel.edu/~mccoy/courses/cisc4-681.10f/lec-materials/handouts/search-water-jug-handout.pdf)
2. [math.tamu.edu](http://www.math.tamu.edu/~dallen/hollywood/diehard/diehard.htm)

I also had to do the same problem in my own college days. At the time I was learning python and thought it might be an interesting problem to solve with [python](https://github.com/vigneshsarma/water-jug). Looking at the code now I feel embarrassed. Am not even sure how I got it working. Rewriting it in python seems rather uninteresting now. Rather using a static functional language like Haskell seamed to make it more interesting. Obviously once you look at the code you will realize currently I don't know much of Haskell either.

Rather than trying to solve the problem in one shot, I decided to limit the initial version some what.

## Updated Problem

There can be some Water Jars, with any given capacity. The challenge will be to start from that to reach the given final state which will be get some specified quantity of water in these Jars. These Jugs have no measurements. But you know the full capacity of the Jugs.

As a starting point max number of Jars will be limited to two.

## Data

One of the most interesting things about writing in a language like Haskell is how Types and data become powerful design tools. So to start with let us define `Jug` and `State`  which is the combination of `Jug`s.

```haskell

-- WaterJug.hs

--  Jug capacity, holding
data Jug = Jug Int Int deriving (Show, Eq, Ord)

-- State left right
data State = State Jug Jug deriving (Show, Eq, Ord)

-- Problem initial state, destination state
data Problem = Problem State State deriving (Show, Eq, Ord)

-- StateMap, type alias
type StateMap = M.Map State [State]

emptyJug :: Int -> Jug
emptyJug c = Jug c 0

newProblem :: Int -> Int -> Int -> Int -> Problem
newProblem rc lc r l = Problem (State (emptyJug rc) (emptyJug lc)) (State (Jug rc r) (Jug lc l))
```

The `deriving` ensures that we can have reasonable string representation for these objects, they can be equated to each other, ordered etc.

All these types we created using data are tuples. They can only contain two arguments. They are also positional.

`emptyJug` function can be used to create an empty `Jug` of any given capacity.

`newProblem` function can be used to define a problem which include jug capacities and the final state we want to achieve.

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

Now given a starting state and final state, we try to find all the possible states between them. The `allStates` function is a kind of wrapper for the inner `allStates'` function. `allStates` calls the inner functions with a correct set of initial arguments, which implements most of the logic.

``` haskell
allStates :: Problem -> StateMap
allStates (Problem i f) =
  let
    allStates' :: State -> S.Set State -> StateMap -> StateMap
    allStates' current queue m
      | S.null queue = m
      | otherwise = allStates' next queue'' m'
      where
        ns = getNextState current
        -- insert current state and its possible transitions to the StateMap
        m' = M.insert current ns m
        queue' = S.delete current queue
        -- filter out all the states that have already been visited.
        new_q = S.fromList $ filter (\s -> M.notMember s m) ns
        -- if final state is in one of these, we dont care for other transitions
        -- from that point
        queue'' = if S.member f new_q
          then queue'
          else S.union queue' new_q
        next = head $ S.toList queue''
  in
    allStates' i (S.fromList [i]) M.empty
```

You can think of the `where` clauses as being executed top to bottom. In reality though they are all lazy and could be executed as and when the need arises. So order has no meaning.

## Is there a possible path from initial to final state?

Once we have the `StateMap` we should be able to find if there is a possible path from initial to final state. For that we flatten out the values of `StateMap` and check if the final state is one of `State`.

``` haskell
isPathPossible :: Problem -> StateMap -> Bool
isPathPossible (Problem _ f) m = S.member f $ S.fromList $ concat $ M.elems m
```

## Find Paths from Initial to Final State

Once we know there is a path, we walk the `StateMap` depth first to find possible paths to the final state. We look at this like a tree with initial state as the root node. When we find a path that reaches the final state we add that to possible paths and continue with our search. One of the optimizations we do is, if we find cycles, ie same state being repeated we end our search on that branch. Similarly if a state is not in `StateMap` once again we end our search on that branch. In all other cases we fold over next states and call `findPaths'` recursively. `findPaths` and `findPaths'` are similar to `allStates` and `allStates'` in that `findPaths` is mostly a limited wrapper over the inner function which implements most of the logic.

``` haskell
findPaths :: Problem -> StateMap -> [[State]]
findPaths (Problem i f) m =
  findPaths' [i] []
  where
    findPaths' :: [State] -> [[State]] -> [[State]]
    findPaths' (x:xs) ps
      -- we have found a cycle, return the routes we have found
      -- and end the search on this branch
      | x `elem` xs = ps
      -- found a full path, add that to routes and end branch
      | x == f  = (reverse $ x:xs):ps
      | otherwise = case M.lookup x m of
                      Nothing -> ps
                      Just ls -> foldl (\ps' l-> findPaths' (l:x:xs) ps') ps ls
```

## Shortest of the Paths

Now that we have a bunch of candidates for paths to final state, its time to find the shortest one. For that we just compare the length of each of the paths like so:

``` haskell
shortestPath :: [[State]] -> [State]
shortestPath ps = foldl (\a x -> if (length a) > (length x)
                            then x else a) (head ps) (tail ps)
```

## Putting it all together

Now we have all the pieces to solve a given `Problem`.

``` haskell
solve :: Problem -> Maybe [State]
solve p = if isPathPossible p ss
          then Just $ shortestPath $ findPaths p ss
          else Nothing
  where
    ss = allStates p
```

The returned type is a bit interesting, it says given a `Problem` maybe we have a solution to get from initial to final state. :)

## Conclusion

To conclude for now let me present a simple REPL session.

``` haskell
> :l WaterJug.hs
> solve $ newProblem 4 3 2 2
Nothing
>
> solve $ newProblem 5 3 4 0
Just [State (Jug 5 0) (Jug 3 0),State (Jug 5 5) (Jug 3 0),State (Jug 5 2) (Jug 3 3),State (Jug 5 2) (Jug 3 0),State (Jug 5 0) (Jug 3 2),State (Jug 5 5) (Jug 3 2),State (Jug 5 4) (Jug 3 3),State (Jug 5 4) (Jug 3 0)]
>
> solve $ newProblem 4 3 2 0
Just [State (Jug 4 0) (Jug 3 0),State (Jug 4 0) (Jug 3 3),State (Jug 4 3) (Jug 3 0),State (Jug 4 3) (Jug 3 3),State (Jug 4 4) (Jug 3 2),State (Jug 4 0) (Jug 3 2),State (Jug 4 2) (Jug 3 0)]
>
```

Next time we will try to solve this for `n` jugs.
