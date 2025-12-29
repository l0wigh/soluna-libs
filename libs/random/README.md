# Random Library

Some PRNG functions. Nothing too fancy.

## Functions

- `(rand seed)` -> `(random-number, (rand next-seed))` where the second argument is a function that you can call.
- `(rand-gen seed)` -> Function used to generate the next rand function call.
- `(rand-next randlist)` -> returns the next (random-number, (rand next-seed)) pair based on the one you gives.
- `(rand-unix)` -> returns a random number based on the current unix time.
