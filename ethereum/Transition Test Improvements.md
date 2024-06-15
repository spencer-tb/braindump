
## Existing transition tests

Currently within EEST we only support basic transition test functionality.

We define a transition fork as like so:
```python=
@transition_fork(to_fork=Prague, at_timestamp=15_000)
class CancunToPragueAtTime15k(Cancun):
    """
    Cancun to Prague transition at Timestamp 15k
    """
    pass
```

This allows us to mark tests like the following,
```python=
@pytest.mark.valid_at_transition_to("Cancun")
def test_beacon_root_contract_deploy(
...
```
such that they are `fill`'d with the with the transition in mind. In this test case Prague is activated at a timestamp of 15k.

However, we must note that these tests are written with the latter in mind, and are extremely specific to the features before and after the transition.


## Using our existing tests during a fork transition

The existing transition tests, and markers are indeed useful, but currently they do not provide the support to fill all of our existing (non-transition) tests during a transition to another fork.

Please check the following issue, regarding the implementation.
https://github.com/ethereum/execution-spec-tests/issues/611

The simple approach would work in such a way where the genesis block would start on the pre-transition (Cancun) fork, and the next block would the first block from the test at the post-transition (Prague) fork.

If we were filling for Cancun, this would additionally fill all non-transition Cancun tests during the transition to Prague. Again where the transition occurs on the *first* block.

To achieve this we can use the existing transition base class method `always_execute()`:
```python=
@classmethod
def always_execute(cls) -> bool:
    """
    Whether the transition fork should be treated as a normal fork and all tests should
    be filled with it.
    """
    raise Exception("Not implemented")
```

This would provide additional coverage during the transition, essentially for free.

## Extending transition forks for Verkle

When testing the transition for Verkle I would ideally like to go overkill. There are so many corner cases that could crop up so why not go full throttle and generate and obscenely large amount of transition tests.

There primarily 3 different ways we should test the transtion. Initially with the Verkle stride (num. key/values transitioned from MPT to VKT) set to zero.

- **Pre-fork transition tests:** Here we execute all the test blocks before the transition, where the MPT is slowly filled up (during Shanghai). At the point of transition (Verkle fork), the MPT is converted to VKT.
    - Will require the fork transition time to be >32 - probably 15k, assuming no test has >1250 blocks we can generate all blocks for a test before the transition timestamp (assuming 12 seconds between each block).
- **Mid-fork transition tests:** This is essentially the case presented initially, where the first block of the test starts at the transition fork timestamp. If the timestamp is set to 15k, then the this will be the timestamp of the first block.
    -  The latter however, is the naive approach. Ideally we do this for each block. For example if a test has 4 blocks, we would generate 4 separate mid transition tests. Where the transition block occurs on each different block for each of these tests.
-  **Post-fork transition tests:** Here we start with some initial MPT. This can be anything from random to determinstic, crazy large or reasonable in size.  Once the transition from MPT to VKT is complete, we then fill the test blocks.
    -  For zero stride, this can simply be the genesis alloc starting at Shanghai, and the test blocks can start on the first block after the fork transition block.


### Building on top of the Verkle transition tests

If the above sounds complicated, we need to again extend these. For each of these Verkle transition tests a co-variant matrix can be created and applied to each test such that we parameterize the test generation. This means we will have at least 12 fork-transition tests for each EEST non-transition test.

Lets define the following single col. matrices:

- Transition Type (T):
    - Pre-fork transition
    - Mid-fork transition
    - Post-fork transition

- Verkle Stride (V):
    - Zero stride (num. key/values transitioned from MPT to VKT is zero)
    - With stride (num. key/values transitioned is greater than zero)

- Initial MPT State (M):
    - Zero initial MPT (empty MPT)
    - With initial MPT (non-empty MPT, random or deterministic)

Where our covariant matrix for test parameterizaion C:

- C = T x V x M

Note that the stride and MPT size (including the accounts from test blocks executed pre fork transition), will be correlated. When stride is enabled we will need to be able to generate dummy blocks to fully finish a test such the Verkle transition completes.

## Verkle transition EEST testing phases

For each phase we will increase the complexity of testing and thus type of implementation. Geth must successfully fill and execute all these tests, as well as another client passing all the generated tests before moving onto the next stage.

Each stage will include a specific release, that other clients can use for testing to catch up. These will be versioned and updated if bugs are found.

1a) Naive mid-fork transition tests. In-progress now. No stride enabled. No additional init MPT. Here we create a single fork-transition test, by setting the first block of each non-transition test to the timestamp of the fork transition. Simplest Verkle fork transition sanity check.

1b) Naive mid-fork with init MPT. Same as before with a large init MPT. No stride still.

2a) Full Mid-fork transition tests. Same as 1a) but with all block versions, i.e where there is an individual test, for each test block on the fork transition block.

2b) Full Mid-fork transition tests with init MPT. Same as 2a) but with large init MPT. No stride still.

TODO still. 

## Updating the framework for multi-transition tests

### How do we achieve this?

To me the obvious answer is to create a verkle specific transition class marker that can be applied to forks defined within `transition.py`:

```python=
@verkle_transition_fork(
    to_fork=ShanghaiEIP6800,
    at_timestamp=15_000,
    always_execute=True,
    pre_fork=True,
    mid_fork=True,
    post_fork=True,
    init_mpt_accounts=1000,
    stride_enabled=False,
)
class ShanghaiToShanghaiEIP6800AtTime15k(Shanghai):
    """
    Shanghai to Verkle transition at Timestamp 15k
    """
    pass
```


Where we keep the original `@transition_fork` marker. Such that:
```
always_execute=True,
```

Can be used to generate the non-naive mid-fork transition tests for all existing forks. Not just Verkle. This would somewhat be useful now.

Its unlikely we'll need to use the Verkle transition changes in a post Verkle world.
