Making Pseudonyms in R
================

In this document we will show how to make pseudonyms in R. The methods
presented here are discussed in the ENISA report “[Pseudonymisation
techniques and best
practices](https://www.enisa.europa.eu/publications/pseudonymisation-techniques-and-best-practices)”.

To begin with, we need a data set, so we construct one.

``` r
dataset <- data.frame(
  names = c("Betty Davis", "Peggy Sue", "Frank Zappa"),
  dob = c("1944-07-26", "1957-09-20", "1940-12-21"),
  gender = c("F", "F", "M"),
  v1 = sample.int(50, 3),
  v2 = sample.int(50, 3),
  v3 = sample.int(70, 3)
) 
```

Now we have a data set containing a number of demographics (names, date
of birth, gender) and some research data (v1-v3).

``` r
print(dataset)
```

            names        dob gender v1 v2 v3
    1 Betty Davis 1944-07-26      F 19 15 24
    2   Peggy Sue 1957-09-20      F  7 11  5
    3 Frank Zappa 1940-12-21      M 32 29 58

We will now use 3 different methods to create pseudonyms for this data:

1.  counters
2.  random number generators
3.  hashing

Counter and RNG pseudonyms can be made before the actual data
collection. Hashing pseudonyms can be made when

# Counter pseudonyms

For this method, we want to create pseudonyms of the form
`<prefix><number>`. We do this by creating a series of numbers, and
combine it with a predefined prefix (e.g. “PP”). If you make the
pseudonyms before the data are collected, make sure to create enough.

``` r
prefix <- "PP"                # The prefix we want to use 
n1 <- 1:999                   # Make a series of integers. 
n2 <- sprintf("%03d", n1)     # Add leading zero(s) to make them equal length
n3 <- paste0(prefix, n2)      # Add prefix
```

Once we have done that, it’s just a matter of combining the data columns
from the original data set with the newly created “id”-column containing
the counter pseudonyms.

``` r
counter <- dataset[,4:6]
counter$id <- n3[1:nrow(counter)] 
dataset$id <- counter$id
```

Presto! We have now a pseudonymized data set:

``` r
print(counter)
```

      v1 v2 v3    id
    1 19 15 24 PP001
    2  7 11  5 PP002
    3 32 29 58 PP003

And a keyfile:

``` r
print(dataset)
```

            names        dob gender v1 v2 v3    id
    1 Betty Davis 1944-07-26      F 19 15 24 PP001
    2   Peggy Sue 1957-09-20      F  7 11  5 PP002
    3 Frank Zappa 1940-12-21      M 32 29 58 PP003

# Randon Number Generator (RNG) pseudonyms

Creating an RNG pseudonym is similar to creating a counter pseudonym,
except that now the list with pseudonyms is randomized.

``` r
n1 <- 1:999                       # Make a series of integers
n2 <- sprintf("%03d", n1)         # Beautify the numbers with leading zero
n3 <- paste("PP", n2, sep = "")   # Add prefix
n4 <- sample(n3)                  # Randomize the order of the pseudonyms
```

Again, we create a pseudonomyzed data set, as well as a keyfile

``` r
rng <- dataset[,4:6]
rng$id <- n4[1:nrow(rng)] 
dataset$id <- rng$id
```

We have now a pseudonymized data set:

``` r
print(rng)
```

      v1 v2 v3    id
    1 19 15 24 PP720
    2  7 11  5 PP076
    3 32 29 58 PP282

And the keyfile:

``` r
print(dataset)
```

            names        dob gender v1 v2 v3    id
    1 Betty Davis 1944-07-26      F 19 15 24 PP720
    2   Peggy Sue 1957-09-20      F  7 11  5 PP076
    3 Frank Zappa 1940-12-21      M 32 29 58 PP282

# Hashed pseudonyms (with secret key)

In addition to counters and random number generators, hashing is also
used in some cases to construct pseudonyms. Hashing is a process in
which information (i.e. a combination of personal variables) is
converted into a unique fixed-length code (the “hash” or “hash code”)
using a hashing algorithm.

What is unique about hashing is that it only works in one direction. You
can transform information into a hash code via the hash function, but
you cannot transform the hash code back to the original information.

Pseudonyms made by hashing are sometimes preferred because they allow
you to generate the pseudonyms from the data, on the fly, meaning that
in principle, no keyfile is necessary. You only need to know how the
hashes are created and the key/secret that was used. For instance in
longitudinal setups, this can be an advantage.

However, if not done properly, there is also a major threat: If no key
is used while hashing it is possible to “guess” hash codes by hashing
combinations of personal data and comparing the resulting codes.
**Hashing without a key is therefore unsuitable for pseudonymization**.
To securely generate hash codes, using a secret ‘key’ is indispensable.

## Step by step guide

In this example we use the `openssl` library for hashing, but the same
results can be accomplished using the `digest` library.

``` r
library(openssl)
```

We start by choosing a key, which will be used as a kind of password
while hashing. The key should be kept secret. It is therefore important
to remove the key from your code from the moment that the pseudonyms are
created. If you want to keep the key safe and failing memory proof, put
it in a password manager for later use.

``` r
MySecret = "this is the secret Key which you should REMOVE from you code"
```

The next step is choosing the personal data item you want to use to
derive the hash codes from. Of course, for each person in the data set,
this combination should be unique. Therefore, we chose to paste together
the name, date of birth and gender into one long string.

``` r
tmp <- paste0(dataset$names, dataset$dob, dataset$gender)   # paste together personal data, create a unique string
```

Once we have done that, we pull these string through the hash-function
(`sha2` from the `openssl`-library in this case) also using the secret
key defined earlier.

``` r
hashes <- sha2(tmp, key = MySecret)                         # hash it
```

In this step, we truncate (shorten) the resulting hash code in order to
make them more manageable. This of course is an optional step. It is no
problem at all to use the long hash codes. Even more, if you have **very
big data sets** (say thousands of entries), **you should not truncate
the hash codes** to avoid the risk of identical hash codes
(i.e. collisions).

``` r
hashes8 <- substr(hashes, 1, 8)      # Optionally truncate long hashes to something more workable. 
                                     # NOT for big data sets! 
```

Once the hashes are ready, we can create the pseudonymized data set
containing the research data and the hash id’s.

``` r
hashed <- dataset[,4:6]
hashed$id <- hashes8
```

This is the resulting dataset:

``` r
print(hashed)
```

      v1 v2 v3       id
    1 19 15 24 19cc9d2c
    2  7 11  5 17e3b7c9
    3 32 29 58 e24d4874

Remember that it is not necessary to add the hash codes to the original
data set. Instead, we have documented our method (with this code), and
we can reveal the link between the data and the identities of the
participants whenever it is necessary. In other words, it is no longer
necessary to keep the research data available together with the personal
data used to create the hashes.

Therefore, we only keep the data necessary to re-create the hash codes
when necessary:

``` r
keyfile <- subset(dataset, select = c(names, dob, gender))
```

and throw away the data set containing both the personal data and
research data.

``` r
rm(dataset)
```

Finally, we have two data sets left:

The pseudonymized data set containing the research data and the hash
codes:

``` r
print(hashed, comment=NA)
```

    ##   v1 v2 v3       id
    ## 1 19 15 24 19cc9d2c
    ## 2  7 11  5 17e3b7c9
    ## 3 32 29 58 e24d4874

And the “keyfile” containing the variables used to create the hash
codes:

``` r
print(keyfile, comment=NA)
```

    ##         names        dob gender
    ## 1 Betty Davis 1944-07-26      F
    ## 2   Peggy Sue 1957-09-20      F
    ## 3 Frank Zappa 1940-12-21      M

As the keyfile contains personal information, it should be kept in a
safe place.
